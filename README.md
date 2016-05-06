# Wifi-Direct

# Titulo1
## Titulo2
###### Titulo3
Esto es texto normal y puedo escribir lo que quiera
**This is bold text**
*This text is italicized*
```java
public class Hello {
  public static void main(String[] args) {
    System.out.println("Hola mundo");
  }
}
```
## Guía básica:
#### 2) Creación del BroadcastReceiver: <br/>
Se crea una clase java con el nombre de WiFiDirectBroadcastReceiver que será un subclase de BroadcastReceiver, este componente esta destinado a recibir y responder a los siguientes eventos: <br/>
*Activación o desactivación de Wi-Fi Direct <br/>
*La lista de dispositivos(peers) ha cambiado.<br/>
*El estado de la conectividad Wi-Fi Direct peer-to-peer ha cambiado.<br/>
*La información del dispositivo ha cambiado.
Esto se verá ahora con mayor detalle.
```java
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.net.NetworkInfo;
import android.net.wifi.p2p.WifiP2pDevice;
import android.net.wifi.p2p.WifiP2pManager;
import android.net.wifi.p2p.WifiP2pManager.Channel;
import android.net.wifi.p2p.WifiP2pManager.PeerListListener;
import android.util.Log;

/**
 * A BroadcastReceiver that notifies of important wifi p2p events.
 */
public class WiFiDirectBroadcastReceiver extends BroadcastReceiver {

    private WifiP2pManager manager;
    private Channel channel;
    private WiFiDirectActivity activity;

    /**
     * @param manager WifiP2pManager system service
     * @param channel Wifi p2p channel
     * @param activity activity associated with the receiver
     */
    public WiFiDirectBroadcastReceiver(WifiP2pManager manager, Channel channel,
            WiFiDirectActivity activity) {
        super();
        this.manager = manager;
        this.channel = channel;
        this.activity = activity;
    }

    /*
     * (non-Javadoc)
     * @see android.content.BroadcastReceiver#onReceive(android.content.Context,
     * android.content.Intent)
     */
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if (WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION.equals(action)) {

            // UI update to indicate wifi p2p status.
            int state = intent.getIntExtra(WifiP2pManager.EXTRA_WIFI_STATE, -1);
            if (state == WifiP2pManager.WIFI_P2P_STATE_ENABLED) {
                // Wifi Direct mode is enabled
                activity.setIsWifiP2pEnabled(true);
            } else {
                activity.setIsWifiP2pEnabled(false);
                activity.resetData();

            }
            Log.d(WiFiDirectActivity.TAG, "P2P state changed - " + state);
        } else if (WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION.equals(action)) {

            // request available peers from the wifi p2p manager. This is an
            // asynchronous call and the calling activity is notified with a
            // callback on PeerListListener.onPeersAvailable()
            if (manager != null) {
                manager.requestPeers(channel, (PeerListListener) activity.getFragmentManager()
                        .findFragmentById(R.id.frag_list));
            }
            Log.d(WiFiDirectActivity.TAG, "P2P peers changed");
        } else if (WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION.equals(action)) {

            if (manager == null) {
                return;
            }

            NetworkInfo networkInfo = (NetworkInfo) intent
                    .getParcelableExtra(WifiP2pManager.EXTRA_NETWORK_INFO);

            if (networkInfo.isConnected()) {

                // we are connected with the other device, request connection
                // info to find group owner IP

                DeviceDetailFragment fragment = (DeviceDetailFragment) activity
                        .getFragmentManager().findFragmentById(R.id.frag_detail);
                manager.requestConnectionInfo(channel, fragment);
            } else {
                // It's a disconnect
                activity.resetData();
            }
        } else if (WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION.equals(action)) {
            DeviceListFragment fragment = (DeviceListFragment) activity.getFragmentManager()
                    .findFragmentById(R.id.frag_list);
            fragment.updateThisDevice((WifiP2pDevice) intent.getParcelableExtra(
                    WifiP2pManager.EXTRA_WIFI_P2P_DEVICE));

        }
    }
}

```
Se puede observar que la clase dispone de tres atributos:<br/>
**WifiP2pManager manager:** Es la encargada de gestionar la conectividad de Wi-Fi Direct peer-to-peer, esto significa que tiene como responsabilidades: Encontrar los dispositivos disponibles para un conexión y configurar la conexión especifica a un peer.<br/>
**Channel channel:**Es el conducto que permite establecer la comunicación por medio de un servicio I/O, para este caso se utilizará un socket para dicha conexión.<br/>
**WiFiDirectActivity activity:** Es la referencia a la actividad principal de la app, esto se hace con el objetivo de actualizar la actividad principal dependiendo las acciones que realice el BroadcastReceiver.<br/>






