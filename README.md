package com.example.myapplication

import android.Manifest
import android.app.Activity
import android.app.AlertDialog
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothDevice
import android.bluetooth.BluetoothGatt
import android.bluetooth.BluetoothGattCallback
import android.bluetooth.BluetoothManager
import android.bluetooth.BluetoothProfile
import android.bluetooth.le.BluetoothLeScanner
import android.bluetooth.le.ScanCallback
import android.bluetooth.le.ScanFilter
import android.bluetooth.le.ScanResult
import android.bluetooth.le.ScanSettings
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Build
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.util.Log
import android.widget.Button
import androidx.activity.result.contract.ActivityResultContracts
import androidx.annotation.RequiresApi
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView


class MainActivity : AppCompatActivity() {

    private lateinit var scanButton : Button
    private final val PERMISSION_REQUEST_CODE = 100

    private val scanResults = mutableListOf<ScanResult>()
    private lateinit var recyclerView : RecyclerView
    private lateinit var deviceAdapter: bleDeviceAdapter




    private var isScanning = false
        set(value){
            field = value
            runOnUiThread {
                scanButton.text = if(value) "Stop Scan" else "Start Scan"
            }
        }

    private val handler = Handler(Looper.getMainLooper())

    private val bluetoothAdapter : BluetoothAdapter by lazy {
        val bluetoothManager = getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
        bluetoothManager.adapter
    }

    private val bleScanner : BluetoothLeScanner by lazy {
        bluetoothAdapter.bluetoothLeScanner
    }

    private val settings = ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build()

    val filters = listOf(
        ScanFilter.Builder()
            .setDeviceName("Your Device Name")
            .build()
    )

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // recycler view binding with adapter

        recyclerView = findViewById(R.id.recyclerView)
        deviceAdapter = bleDeviceAdapter(scanResults, this)
        recyclerView.adapter = deviceAdapter
        recyclerView.layoutManager = LinearLayoutManager(this)
        
        deviceAdapter.setOnItemClickListener(object : bleDeviceAdapter.onItemClickListener {
            override fun onItemClick(position: Int) {
                    connecttodevice(position)
                }

        })




        scanButton = findViewById<Button>(R.id.btn_start_scan)

        scanButton.setOnClickListener{
            if(isScanning)
                stopBleScan()
            else
                startBleScan()
        }
        if(!bluetoothAdapter.isEnabled)
            promptEnableBluetooth()
    }



    private fun startBleScan() {
        if(!hasRequiredBlueetoothPermissions()) {
            requestRelevantRunTimePermissions()
        }
        else{
            scanResults.clear()
            deviceAdapter.notifyDataSetChanged()
            if(ActivityCompat.checkSelfPermission(this,Manifest.permission.BLUETOOTH_SCAN) != PackageManager.PERMISSION_GRANTED)
                return
            bleScanner.startScan(null,settings,scanCallback)
            isScanning = true

            Handler(mainLooper).postDelayed({
                stopBleScan()
            },10000) // after 10 sec stop scanning


        }
    }

    private fun stopBleScan() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_SCAN) != PackageManager.PERMISSION_GRANTED) {
            return
        }
        bleScanner.stopScan(scanCallback)
        isScanning = false
    }

    private fun Context.hasPermission(permissionType : String) : Boolean {
        return ContextCompat.checkSelfPermission(this,permissionType) == PackageManager.PERMISSION_GRANTED
    }

    private fun Context.hasRequiredBlueetoothPermissions (): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            hasPermission(Manifest.permission.BLUETOOTH_SCAN) &&
                    hasPermission(Manifest.permission.BLUETOOTH_CONNECT)
        } else {
            hasPermission(Manifest.permission.ACCESS_FINE_LOCATION)
        }
    }

    private fun Activity.requestRelevantRunTimePermissions() {
        if(hasRequiredBlueetoothPermissions())
        {return}
        when{
            Build.VERSION.SDK_INT < Build.VERSION_CODES.S -> {
                requestLocationPermission()
            }
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
                requestBluetoothPermission()
            }
        }
    }

    private fun requestLocationPermission() = runOnUiThread(){
        AlertDialog.Builder(this)
            .setTitle("Location Permission required")
            .setMessage(
                "Starting from Android M (6.0), the system requires apps to be granted " +
                        "loaction access in order to scan for BLE devices."
            )
            .setCancelable(false)
            .setPositiveButton(android.R.string.ok){ _,_->
                ActivityCompat.requestPermissions(
                    this,
                    arrayOf(Manifest.permission.ACCESS_FINE_LOCATION),
                    PERMISSION_REQUEST_CODE
                )

            }
            .show()

    }

    @RequiresApi(Build.VERSION_CODES.S)
    private fun requestBluetoothPermission () = runOnUiThread() {
        AlertDialog.Builder(this)
            .setTitle("Bluetooth Permission Required")
            .setMessage(
                "Starting from Android 12, the system requires apps to be granted " +
                        "Bluetooth access in order to scan for BLE devices."
            )
            .setCancelable(false)
            .setPositiveButton(android.R.string.ok) { _,_ ->
                ActivityCompat.requestPermissions(
                    this,
                    arrayOf(
                        Manifest.permission.BLUETOOTH_SCAN,
                        Manifest.permission.BLUETOOTH_CONNECT
                    ),
                    PERMISSION_REQUEST_CODE

                )
            }
            .show()
    }


    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if(requestCode != PERMISSION_REQUEST_CODE) return

        val containsPermanentDenial = permissions.zip(grantResults.toTypedArray()).any {
            it.second == PackageManager.PERMISSION_DENIED &&
                    !ActivityCompat.shouldShowRequestPermissionRationale(this,it.first)
        }

        val containsDenial = grantResults.any{it == PackageManager.PERMISSION_DENIED}
        val allGranted = grantResults.all{it == PackageManager.PERMISSION_GRANTED}

        when {
            containsPermanentDenial -> {
                // handle permanent denial case
            }
            containsDenial -> {
                requestRelevantRunTimePermissions()
            }
            allGranted && hasRequiredBlueetoothPermissions() -> {
                startBleScan()
            }
            else -> {
                recreate()
            }
        }

    }



    private val bluetoothEnablingResult = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if(result.resultCode == Activity.RESULT_OK){
            // bluetooth is enabled
        }
        else{
            // user dismissed or denied Bluetooth Prompt
            promptEnableBluetooth()
        }
    }

    override fun onResume() {
        super.onResume()
        if(!bluetoothAdapter.isEnabled)
            promptEnableBluetooth()
    }

    private fun promptEnableBluetooth() {
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && !hasPermission(Manifest.permission.BLUETOOTH_CONNECT)) {
            // insufficient permission to prompt for bluetooth enabling
            return
        }
        if(!bluetoothAdapter.isEnabled){
            Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE).apply{
                bluetoothEnablingResult.launch(this)
            }
        }

    }



    // performing BLE Scan and showing the result in log-cat

    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult?) {

            val indexQuery = scanResults.indexOfFirst { it.device.address == result?.device?.address}
            if(indexQuery !=-1){
                // scan result already present with same result

                scanResults[indexQuery] = result!!
                deviceAdapter.notifyItemChanged(indexQuery)
            }
            else {
                with(result!!.device) {
                    Log.i("ScanCallBack", "Found BlE device : Name ${name ?: "Unnamed"}, Adress: ${address}")
                }
                scanResults.add(result)
                deviceAdapter.notifyItemInserted(scanResults.size-1)
            }
        }

        override fun onScanFailed(errorCode: Int) {
            Log.e("ScanCallBack", "Scan falied with error: $errorCode")
        }
    }

    // here device is the device which i will selefor this from the list of all devices
    

    fun connecttodevice(device: BluetoothDevice){
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_SCAN) != PackageManager.PERMISSION_GRANTED) {
            return
        }
        val bluetoothGatt: BluetoothGatt = device.connectGatt(this,false, gattCallback)
    }


    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
            val deviceAdress = gatt?.device?.address
            if(status == BluetoothGatt.GATT_SUCCESS) {
                if(newState == BluetoothProfile.STATE_CONNECTED){
                    Log.w("BluetoothGattCallback" , "Sucessfully connected to $deviceAdress")

                } else if(newState == BluetoothProfile.STATE_DISCONNECTED){
                    Log.w("BluetoothGattCalback", "Successfully disconnected from $deviceAdress")
                }
                else{
                    Log.w("BluetoothGattCallback", "Error $status encontered for $deviceAdress")
                    gatt?.close()
                }
            }
        }
    }


}




package com.example.myapplication

import android.bluetooth.le.ScanResult
import android.content.Context
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.AdapterView.OnItemClickListener
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class bleDeviceAdapter(private val devices : List<ScanResult>, var context: Context) : RecyclerView.Adapter<bleDeviceAdapter.ViewHolder>() {

    private lateinit var myListener: OnItemClickListener
    interface onItemClickListener{
        fun onItemClick(position: Int)
    }
    fun setOnItemClickListener(listener: onItemClickListener){
        myListener = listener
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.device_row_layout, parent, false)

        return ViewHolder(view, myListener)
    }

    override fun getItemCount(): Int {
        return devices.size

    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {

        val result = devices[position]
        holder.deviceName.text = result.device.name
        holder.deviceAddress.text = result.device.address
        // add onClickListener to the item


    }



    class ViewHolder(itemView: View, listener: OnItemClickListener) : RecyclerView.ViewHolder(itemView){
        val deviceName: TextView = itemView.findViewById(R.id.deviceNameTextView)
        val deviceAddress :TextView = itemView.findViewById(R.id.deviceAdressTextView)

        init {
            itemView.setOnClickListener {
                listener.onItemClick(adapterPosition)
            }
        }

    }


}
