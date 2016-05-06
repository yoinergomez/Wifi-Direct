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
####- Creación del BroadcastReceiver: <br/>
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
```WifiP2pManager manager:``` Es la encargada de gestionar la conectividad de Wi-Fi Direct peer-to-peer, esto significa que tiene como responsabilidades: Encontrar los dispositivos disponibles para un conexión y configurar la conexión especifica a un peer.<br/>
```Channel channel:```Es el conducto que permite establecer la comunicación por medio de un servicio I/O, para este caso se utilizará un socket para dicha conexión.<br/>
```WiFiDirectActivity activity:``` Es la referencia a la actividad principal de la app, esto se hace con el objetivo de actualizar la actividad principal dependiendo las acciones que realice el BroadcastReceiver.<br/>

El método ```onReceive(Context context, Intent intent)``` captuta la acción asociada al intento que invocó el BroadcastReceiver.<br/>
Cuando se genera el evento Activación o desactivación de Wi-Fi Direct simplemente se actualiza ese estado a la atividada principal. <br/>
```java
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
  }
```
Cuando la lista de dispositivos(peers) disponibles ha cambiado, se actualiza el objeto  ```manager``` con la lista actual de peers que se encuentran en un ```ListFragment```
```java
   else if (WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION.equals(action)) {

            // request available peers from the wifi p2p manager. This is an
            // asynchronous call and the calling activity is notified with a
            // callback on PeerListListener.onPeersAvailable()
            if (manager != null) {
                manager.requestPeers(channel, (PeerListListener) activity.getFragmentManager()
                        .findFragmentById(R.id.frag_list));
            }
            Log.d(WiFiDirectActivity.TAG, "P2P peers changed");
  }
```
Cuando el estado de la conectividad del Wi-Fi Direct ha cambiado, se obtiene la información actual de la conexión con la red, en caso de estar conectado el dispositivo puede establecer conexión con algún peer disponible para transferir información,si esto no ocurre el dispositivo no tendrá peers disponibles.
```java
   else if (WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION.equals(action)) {

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
  }

```

Por último, cuando la información de un dispositivo ha cambiado, se actualiza fragmento asociado a este dispositivo.
```java
} else if (WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION.equals(action)) {
            DeviceListFragment fragment = (DeviceListFragment) activity.getFragmentManager()
                    .findFragmentById(R.id.frag_list);
            fragment.updateThisDevice((WifiP2pDevice) intent.getParcelableExtra(
                    WifiP2pManager.EXTRA_WIFI_P2P_DEVICE));
}

```
#### * Creación del DeviceDetailFragment: <br/>
Se crea una clase java con el nombre DeviceDetailFragment que será una subclase de Fragment y además implmentará la interfaz ConnectionInfoListener, esto con el objetivo de conocer la información actual de la conexión establecida entre dos peers. El fragmento contienen la información detallada de un dispositivo que está vinculado a la red </br>
La clase tendrá como variables globales:
```java
public class DeviceDetailFragment extends Fragment implements ConnectionInfoListener {
    protected static final int CHOOSE_FILE_RESULT_CODE = 20;
    private View mContentView = null;
    private WifiP2pDevice device;
    private WifiP2pInfo info;
    ProgressDialog progressDialog = null;
```
Se deben sobrescribir algunos métodos ofrecidos por la clase Fragment.<br/>
El método ```public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)``` además de crear la vista del ```Fragment``` se encarga de capturar los ```onClickListener``` asociados a las botones **Connect, Disconnect y Launch Gallery**
En el caso del botón **Connect** se ajusta la configuración de Wi-Fi Direct especificando la dirección de mi dispositivo y la dirección del dispositivo al que me quiero conectar, luego de esto se tratan de conectar los peers en cuestión.
```java
mContentView.findViewById(R.id.btn_connect).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                WifiP2pConfig config = new WifiP2pConfig();
                config.deviceAddress = device.deviceAddress;
                config.wps.setup = WpsInfo.PBC;
                if (progressDialog != null && progressDialog.isShowing()) {
                    progressDialog.dismiss();
                }
                progressDialog = ProgressDialog.show(getActivity(), "Press back to cancel",
                        "Connecting to :" + device.deviceAddress, true, true
//                        new DialogInterface.OnCancelListener() {
//
//                            @Override
//                            public void onCancel(DialogInterface dialog) {
//                                ((DeviceActionListener) getActivity()).cancelDisconnect();
//                            }
//                        }
                        );
                ((DeviceActionListener) getActivity()).connect(config);

            }
        });
```
En caso del botón **Disconnect** solo se invoca al método ```disconnect()``` que es implementado en la actividad principal.
mContentView.findViewById(R.id.btn_disconnect).setOnClickListener(
                new View.OnClickListener() {
```java
                    @Override
                    public void onClick(View v) {
                        ((DeviceActionListener) getActivity()).disconnect();
                    }
                });
```
En caso del botón **Launch Gallery** se crea un Intent para acceder a la galeria y escoger la imagen que será transferida a otro dispositivo.
```java
mContentView.findViewById(R.id.btn_start_client).setOnClickListener(
                new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {
                        // Allow user to pick an image from Gallery or other
                        // registered apps
                        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
                        intent.setType("image/*");
                        startActivityForResult(intent, CHOOSE_FILE_RESULT_CODE);
                    }
                });
```
