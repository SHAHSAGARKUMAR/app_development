package com.example.myapplication

import android.bluetooth.le.ScanResult
import android.content.Context
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class bleDeviceAdapter(private val devices: List<ScanResult>, private val context: Context) : RecyclerView.Adapter<bleDeviceAdapter.ViewHolder>() {

    private var listener: onItemClickListener? = null

    interface onItemClickListener {
        fun onItemClick(position: Int)
    }

    fun setOnItemClickListener(listener: onItemClickListener) {
        this.listener = listener
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.device_row_layout, parent, false)
        return ViewHolder(view, listener)
    }

    override fun getItemCount(): Int {
        return devices.size
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val result = devices[position]
        holder.deviceName.text = result.device.name ?: "Unnamed"
        holder.deviceAddress.text = result.device.address
    }

    class ViewHolder(itemView: View, private val listener: onItemClickListener?) : RecyclerView.ViewHolder(itemView) {
        val deviceName: TextView = itemView.findViewById(R.id.deviceNameTextView)
        val deviceAddress: TextView = itemView.findViewById(R.id.deviceAdressTextView)

        init {
            itemView.setOnClickListener {
                listener?.onItemClick(adapterPosition)
            }
        }
    }
}
