# Wifi-Direct

## Guía básica
###### Objetivo:
Crear una aplicación android que permita enviar una imagen por medio de Wifi-Direct
<br/><br/>
###### Configuraciones iniciales:
* Crear un nuevo proyecto android con el nombre ```Wifi-Direct```, SDK mínimo ```API 16``` y con el tipo de ```actividad vacía``` con nombre ```WiFiDirectActivity``` y el nombre del layout ```activity_wi_fi_direct``` <br/><br/>

###### Creación del BroadcastReceiver: <br/>
[Se crea una clase java](https://github.com/yoinergomez/Wifi-Direct/blob/master/src/com/example/android/wifidirect/WiFiDirectBroadcastReceiver.java) con el nombre de **WiFiDirectBroadcastReceiver** que será un subclase de BroadcastReceiver, este componente esta destinado a recibir y responder a los siguientes eventos: <br/>
* Activación o desactivación de Wi-Fi Direct <br/>
* La lista de dispositivos(peers) disponibles ha cambiado.<br/>
* El estado de la conectividad Wi-Fi Direct peer-to-peer ha cambiado.<br/>
* La información del dispositivo ha cambiado.
Esto se verá ahora con mayor detalle.
<br/><br/>
Se puede observar que [la clase](https://github.com/yoinergomez/Wifi-Direct/blob/master/src/com/example/android/wifidirect/WiFiDirectBroadcastReceiver.java) dispone de tres atributos:<br/>
```WifiP2pManager manager:``` Es la encargada de gestionar la conectividad de Wi-Fi Direct peer-to-peer, esto significa que tiene como responsabilidades: Encontrar los dispositivos disponibles para un conexión y configurar la conexión especifica a un peer.<br/>
```Channel channel:```Es el conducto que permite establecer la comunicación por medio de un servicio I/O, para este caso se utilizará un socket para dicha conexión.<br/>
```WiFiDirectActivity activity:``` Es la referencia a la actividad principal de la app, esto se hace con el objetivo de actualizar la actividad principal dependiendo las acciones que realice el BroadcastReceiver.<br/>

El método ```onReceive(Context context, Intent intent)``` captuta la acción asociada al intento que invocó el BroadcastReceiver.<br/>
Cuando se genera el evento Activación o desactivación de Wi-Fi Direct simplemente se actualiza ese estado a la actividad principal. <br/>
```java
if (WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION.equals(action)) {
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
            if (manager != null) {
                manager.requestPeers(channel, (PeerListListener) activity.getFragmentManager()
                        .findFragmentById(R.id.frag_list));
            }
            Log.d(WiFiDirectActivity.TAG, "P2P peers changed");
  }
```
Cuando el estado de la conectividad del Wi-Fi Direct ha cambiado, se obtiene la información actual de la conexión con la red, en caso de estar conectado el dispositivo puede establecer conexión con algún peer disponible para transferir información, si esto no ocurre el dispositivo no tendrá peers disponibles.
```java
   else if (WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION.equals(action)) {

            if (manager == null) {
                return;
            }

            NetworkInfo networkInfo = (NetworkInfo) intent
                    .getParcelableExtra(WifiP2pManager.EXTRA_NETWORK_INFO);

            if (networkInfo.isConnected()) {
            
                DeviceDetailFragment fragment = (DeviceDetailFragment) activity
                        .getFragmentManager().findFragmentById(R.id.frag_detail);
                manager.requestConnectionInfo(channel, fragment);
            } else {
                activity.resetData();
            }
  }

```

Por último, cuando la información de un dispositivo ha cambiado, se actualiza fragmento asociado a este dispositivo.
```java
 else if (WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION.equals(action)) {
            DeviceListFragment fragment = (DeviceListFragment) activity.getFragmentManager()
                    .findFragmentById(R.id.frag_list);
            fragment.updateThisDevice((WifiP2pDevice) intent.getParcelableExtra(
                    WifiP2pManager.EXTRA_WIFI_P2P_DEVICE));
}

```
#### Creación del DeviceDetailFragment: <br/>
[Se crea una clase java](https://github.com/yoinergomez/Wifi-Direct/blob/master/src/com/example/android/wifidirect/DeviceDetailFragment.java) con el nombre ```DeviceDetailFragment``` que será una subclase de ```Fragment``` y además implementará la interfaz ```ConnectionInfoListener```, esto con el objetivo de conocer la información actual de la conexión establecida entre dos peers. El fragmento contiene la información detallada de un dispositivo que está vinculado a la red </br>
La clase tendrá como variables globales:
```java
    protected static final int CHOOSE_FILE_RESULT_CODE = 20;
    private View mContentView = null;
    private WifiP2pDevice device;
    private WifiP2pInfo info;
    ProgressDialog progressDialog = null;
```
Se deben sobrescribir algunos métodos ofrecidos por la clase Fragment: .<br/>
* El método ```public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)``` además de crear la vista del ```Fragment``` se encarga de capturar los ```onClickListener``` asociados a las botones **Connect, Disconnect y Launch Gallery**
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
                        );
                ((DeviceActionListener) getActivity()).connect(config);

            }
        });
```
En caso del botón **Disconnect** solo se invoca al método ```disconnect()``` que es implementado en la actividad principal.
```java
mContentView.findViewById(R.id.btn_disconnect).setOnClickListener(
                new View.OnClickListener() {
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

En conclusión, el método ```onCreateView``` queda definido así:
```java
 @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {

        mContentView = inflater.inflate(R.layout.device_detail, null);
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
                        );
                ((DeviceActionListener) getActivity()).connect(config);

            }
        });

        mContentView.findViewById(R.id.btn_disconnect).setOnClickListener(
                new View.OnClickListener() {

                    @Override
                    public void onClick(View v) {
                        ((DeviceActionListener) getActivity()).disconnect();
                    }
                });

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

        return mContentView;
    }
```

* El método ```public void onActivityResult(int requestCode, int resultCode, Intent data)``` se utiliza para recebir el resultado de seleccionar una imagen de la galeria,el método se encarga de obetener la ```Uri``` de la imagen seleccionada y crear un ```Intent```, donde se envia la Uri de la imagen, la dirección y el puerto del servidor para iniciar el servicio llamado ```FileTransferService``` que se ocupa solamente de transferir la información del dispositivo cliente al dispositivo servidor.
```java
@Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {

        Uri uri = data.getData();
        TextView statusText = (TextView) mContentView.findViewById(R.id.status_text);
        statusText.setText("Sending: " + uri);
        Log.d(WiFiDirectActivity.TAG, "Intent----------- " + uri);
        Intent serviceIntent = new Intent(getActivity(), FileTransferService.class);
        serviceIntent.setAction(FileTransferService.ACTION_SEND_FILE);
        serviceIntent.putExtra(FileTransferService.EXTRAS_FILE_PATH, uri.toString());
        serviceIntent.putExtra(FileTransferService.EXTRAS_GROUP_OWNER_ADDRESS,
                info.groupOwnerAddress.getHostAddress());
        serviceIntent.putExtra(FileTransferService.EXTRAS_GROUP_OWNER_PORT, 8988);
        getActivity().startService(serviceIntent);
    }
  ```

#### Creación del FileTransferService: <br/>
  * En este punto se necesita crear una nueva clase de java llamada ```FileTransferService``` que es utilizada para transferir la imagen desde el dispositivo cliente al dispositivo servidor. Para ello, se obtiene el ``` Intent```  por él cual se inicio el servicio, de dicho ``` Intent```  se extrae la ``` Uri```  que contiene la ruta de la imagen, la dirección IP y el puerto del dispositivo servidor para crear el ``` Socket```  asociado el dispositivo cliente , por medio de este se establece la conexión al servidor,se copia la imagen y se manda gracias al ```OutpuStream```.
```java
import android.app.IntentService;
import android.content.ContentResolver;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.util.Log;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;

public class FileTransferService extends IntentService {

    private static final int SOCKET_TIMEOUT = 5000;
    public static final String ACTION_SEND_FILE = "com.example.android.wifidirect.SEND_FILE";
    public static final String EXTRAS_FILE_PATH = "file_url";
    public static final String EXTRAS_GROUP_OWNER_ADDRESS = "go_host";
    public static final String EXTRAS_GROUP_OWNER_PORT = "go_port";

    public FileTransferService(String name) {
        super(name);
    }

    public FileTransferService() {
        super("FileTransferService");
    }

    @Override
    protected void onHandleIntent(Intent intent) {

        Context context = getApplicationContext();
        if (intent.getAction().equals(ACTION_SEND_FILE)) {
            String fileUri = intent.getExtras().getString(EXTRAS_FILE_PATH);
            String host = intent.getExtras().getString(EXTRAS_GROUP_OWNER_ADDRESS);
            Socket socket = new Socket();
            int port = intent.getExtras().getInt(EXTRAS_GROUP_OWNER_PORT);

            try {
                Log.d(WiFiDirectActivity.TAG, "Opening client socket - ");
                socket.bind(null);
                socket.connect((new InetSocketAddress(host, port)), SOCKET_TIMEOUT);

                Log.d(WiFiDirectActivity.TAG, "Client socket - " + socket.isConnected());
                OutputStream stream = socket.getOutputStream();
                ContentResolver cr = context.getContentResolver();
                InputStream is = null;
                try {
                    is = cr.openInputStream(Uri.parse(fileUri));
                } catch (FileNotFoundException e) {
                    Log.d(WiFiDirectActivity.TAG, e.toString());
                }
                DeviceDetailFragment.copyFile(is, stream);
                Log.d(WiFiDirectActivity.TAG, "Client: Data written");
            } catch (IOException e) {
                Log.e(WiFiDirectActivity.TAG, e.getMessage());
            } finally {
                if (socket != null) {
                    if (socket.isConnected()) {
                        try {
                            socket.close();
                        } catch (IOException e) {
                            // Give up
                            e.printStackTrace();
                        }
                    }
                }
            }

        }
    }
}
```
  
  Ahora,volvemos a la clase ```DeviceDetailFragment```y se implementa el método ofrecido por la interfaz ```ConnectionInfoListener```: <br/>
  * El método ```public void onConnectionInfoAvailable(final WifiP2pInfo info)```  permite por medio del objeto ```WifiP2pInfo info``` conocer cual peer es el dispotivo cliente y cual es el dispositivo servidor , de acuerdo a esto si se trata del dispositivo servidor se llama a la clase ```FileServerAsyncTask``` que crea el server socket y en caso de ser un dispositivo cliente se habilita el botón **Launch Galery** para que el cliente pueda seleccionar la imagen.
```java 
@Override
    public void onConnectionInfoAvailable(final WifiP2pInfo info) {
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
        this.info = info;
        this.getView().setVisibility(View.VISIBLE);

        // The owner IP is now known.
        TextView view = (TextView) mContentView.findViewById(R.id.group_owner);
        view.setText(getResources().getString(R.string.group_owner_text)
                + ((info.isGroupOwner == true) ? getResources().getString(R.string.yes)
                        : getResources().getString(R.string.no)));

        // InetAddress from WifiP2pInfo struct.
        view = (TextView) mContentView.findViewById(R.id.device_info);
        view.setText("Group Owner IP - " + info.groupOwnerAddress.getHostAddress());

        if (info.groupFormed && info.isGroupOwner) {
            new FileServerAsyncTask(getActivity(), mContentView.findViewById(R.id.status_text))
                    .execute();
        } else if (info.groupFormed) {
            mContentView.findViewById(R.id.btn_start_client).setVisibility(View.VISIBLE);
            ((TextView) mContentView.findViewById(R.id.status_text)).setText(getResources()
                    .getString(R.string.client_text));
        }

        mContentView.findViewById(R.id.btn_connect).setVisibility(View.GONE);
    }
    
```

* El método ```public void showDetails(WifiP2pDevice device) ``` actualiza el Fragment con la información dada por el objeto ```WifiP2pDevice device```.
``` java
   public void showDetails(WifiP2pDevice device) {
        this.device = device;
        this.getView().setVisibility(View.VISIBLE);
        TextView view = (TextView) mContentView.findViewById(R.id.device_address);
        view.setText(device.deviceAddress);
        view = (TextView) mContentView.findViewById(R.id.device_info);
        view.setText(device.toString());

    }
``` 

* El método ```public void resetViews()``` limpia el Fragment
```java 
    public void resetViews() {
        mContentView.findViewById(R.id.btn_connect).setVisibility(View.VISIBLE);
        TextView view = (TextView) mContentView.findViewById(R.id.device_address);
        view.setText(R.string.empty);
        view = (TextView) mContentView.findViewById(R.id.device_info);
        view.setText(R.string.empty);
        view = (TextView) mContentView.findViewById(R.id.group_owner);
        view.setText(R.string.empty);
        view = (TextView) mContentView.findViewById(R.id.status_text);
        view.setText(R.string.empty);
        mContentView.findViewById(R.id.btn_start_client).setVisibility(View.GONE);
        this.getView().setVisibility(View.GONE);
    }
```  
* Ahora dentro de esta clase crearemos la clase ```FileServerAsyncTask``` que como se dijo anteriormente se encarga de crear el server socket asociado al dispositivo servidor.
```java 
public static class FileServerAsyncTask extends AsyncTask<Void, Void, String> {

        private Context context;
        private TextView statusText;

        public FileServerAsyncTask(Context context, View statusText) {
            this.context = context;
            this.statusText = (TextView) statusText;
        }
```
En el método protected ```void onPreExecute()``` simplemente se informa que la tarea asíncrona va a abrir un socket.
```java 
   @Override
        protected void onPreExecute() {
            statusText.setText("Opening a server socket");
        }
```

Como se trata de una tarea asincrona en el método ```protected String doInBackground(Void... params)``` se especifica la tarea que se realizará en segundo plano y que producirá resultados en el hilo principal de la aplicación . En este método se  crea el server socket asociado al dispositivo servidor y con el método ```serverSocket.accept()``` todo el proceso se bloquea hasta que un dispositivo cliente por medio de otro socket pueda establecer conexión con él, luego de que se establece la comunicación por medio de ```client.getInputStream()```  el servidor obtiene la imagen enviada por el cliente y lo que se hace simplemente es crear un ruta en donde se almacenará la imagen.
```java 
@Override
        protected String doInBackground(Void... params) {
            try {
                ServerSocket serverSocket = new ServerSocket(8988);
                Log.d(WiFiDirectActivity.TAG, "Server: Socket opened");
                Socket client = serverSocket.accept();
                Log.d(WiFiDirectActivity.TAG, "Server: connection done");
                final File f = new File(Environment.getExternalStorageDirectory() + "/"
                        + context.getPackageName() + "/wifip2pshared-" + System.currentTimeMillis()
                        + ".jpg");

                File dirs = new File(f.getParent());
                if (!dirs.exists())
                    dirs.mkdirs();
                f.createNewFile();

                Log.d(WiFiDirectActivity.TAG, "server: copying files " + f.toString());
                InputStream inputstream = client.getInputStream();
                copyFile(inputstream, new FileOutputStream(f));
                serverSocket.close();
                return f.getAbsolutePath();
            } catch (IOException e) {
                Log.e(WiFiDirectActivity.TAG, e.getMessage());
                return null;
            }
        }
```

Luego con el método ```protected void onPostExecute(String result)``` se crea un ```Intent``` para que se pueda visualizar la imagen que el dispositivo cliente envió al dispositivo servidor.
```
 @Override
        protected void onPostExecute(String result) {
            if (result != null) {
                statusText.setText("File copied - " + result);
                Intent intent = new Intent();
                intent.setAction(android.content.Intent.ACTION_VIEW);
                intent.setDataAndType(Uri.parse("file://" + result), "image/*");
                context.startActivity(intent);
            }

        }
  }
```
* Por último se define el método ```public static boolean copyFile(InputStream inputStream, OutputStream out)``` fuera de la clase ```FileServerAsyncTask ``` este método simplemente lee el ```inputStream``` y copia la fila al ```out```.
```java
public static boolean copyFile(InputStream inputStream, OutputStream out) {
        byte buf[] = new byte[1024];
        int len;
        try {
            while ((len = inputStream.read(buf)) != -1) {
                out.write(buf, 0, len);

            }
            out.close();
            inputStream.close();
        } catch (IOException e) {
            Log.d(WiFiDirectActivity.TAG, e.toString());
            return false;
        }
        return true;
    }

}
```
**Creación del DeviceListFragment:** <br/>
Se crea una clase java llamada DeviceListFragment que será un subclase de ```ListFragment``` y además implementa la interfaz ```PeerListListener```, esto se hace para capturar el evento asociado a cuando la lista de peers esta disponible, esto significa que la clase será un ```ListFragment``` que contendrá la lista de peers disponibles que se han descubierto en la red.</br>
``` java
import android.app.ListFragment;
import android.app.ProgressDialog;
import android.content.Context;
import android.content.DialogInterface;
import android.net.wifi.p2p.WifiP2pConfig;
import android.net.wifi.p2p.WifiP2pDevice;
import android.net.wifi.p2p.WifiP2pDeviceList;
import android.net.wifi.p2p.WifiP2pManager.PeerListListener;
import android.os.Bundle;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ArrayAdapter;
import android.widget.ListView;
import android.widget.TextView;

import java.util.ArrayList;
import java.util.List;

/**
 * A ListFragment that displays available peers on discovery and requests the
 * parent activity to handle user interaction events
 */
public class DeviceListFragment extends ListFragment implements PeerListListener {

    private List<WifiP2pDevice> peers = new ArrayList<WifiP2pDevice>();
    ProgressDialog progressDialog = null;
    View mContentView = null;
    private WifiP2pDevice device;

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        this.setListAdapter(new WiFiPeerListAdapter(getActivity(), R.layout.row_devices, peers));

    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        mContentView = inflater.inflate(R.layout.device_list, null);
        return mContentView;
    }

    /**
     * @return this device
     */
    public WifiP2pDevice getDevice() {
        return device;
    }
```

* El método ```private static String getDeviceStatus(int deviceStatus)``` es bastante sencillo ya que solo traduce las contantes de la clase ```WifiP2pDevice``` que se refieren al estado de un dispositivo a un string que es él que se muestra en la interfaz gráfica.
```java  
 private static String getDeviceStatus(int deviceStatus) {
        Log.d(WiFiDirectActivity.TAG, "Peer status :" + deviceStatus);
        switch (deviceStatus) {
            case WifiP2pDevice.AVAILABLE:
                return "Available";
            case WifiP2pDevice.INVITED:
                return "Invited";
            case WifiP2pDevice.CONNECTED:
                return "Connected";
            case WifiP2pDevice.FAILED:
                return "Failed";
            case WifiP2pDevice.UNAVAILABLE:
                return "Unavailable";
            default:
                return "Unknown";

        }
    }
```
* Se sobreescribe el método ```public void onListItemClick(ListView l, View v, int position, long id)``` que provee la clase ```ListFragment```, este metodo se ejecuta cuando se selecciona un dispositivo de la lista de peers disponibles, cuando eso sucede se inicializa la conexión entre el dispositivo del usuario y el dispositivo que escogió.

```java
@Override
    public void onListItemClick(ListView l, View v, int position, long id) {
        WifiP2pDevice device = (WifiP2pDevice) getListAdapter().getItem(position);
        ((DeviceActionListener) getActivity()).showDetails(device);
    }
```
* Luego se crea una clase privada  ```WiFiPeerListAdapter```  que tiene la responsabilidad de mantener la lista de ```WifiP2pDevice```
```java
 private class WiFiPeerListAdapter extends ArrayAdapter<WifiP2pDevice> {

        private List<WifiP2pDevice> items;

        /**
         * @param context
         * @param textViewResourceId
         * @param objects
         */
        public WiFiPeerListAdapter(Context context, int textViewResourceId,
                List<WifiP2pDevice> objects) {
            super(context, textViewResourceId, objects);
            items = objects;

        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            View v = convertView;
            if (v == null) {
                LayoutInflater vi = (LayoutInflater) getActivity().getSystemService(
                        Context.LAYOUT_INFLATER_SERVICE);
                v = vi.inflate(R.layout.row_devices, null);
            }
            WifiP2pDevice device = items.get(position);
            if (device != null) {
                TextView top = (TextView) v.findViewById(R.id.device_name);
                TextView bottom = (TextView) v.findViewById(R.id.device_details);
                if (top != null) {
                    top.setText(device.deviceName);
                }
                if (bottom != null) {
                    bottom.setText(getDeviceStatus(device.status));
                }
            }

            return v;

        }
    }
```

* Luego afuera de la clase ```WiFiPeerListAdapter``` y dentro de la clase ```DeviceListFragment``` se define el método ```public void updateThisDevice(WifiP2pDevice device)``` que actualiza la información del dispositivo que esta manejando el usuario.
```java
public void updateThisDevice(WifiP2pDevice device) {
        this.device = device;
        TextView view = (TextView) mContentView.findViewById(R.id.my_name);
        view.setText(device.deviceName);
        view = (TextView) mContentView.findViewById(R.id.my_status);
        view.setText(getDeviceStatus(device.status));
    }
```
* El método ```public void onPeersAvailable(WifiP2pDeviceList peerList) ``` refresca la lista de peers disponibles luego de que se buscaron dispositivos en la red.
```java
@Override
    public void onPeersAvailable(WifiP2pDeviceList peerList) {
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
        peers.clear();
        peers.addAll(peerList.getDeviceList());
        ((WiFiPeerListAdapter) getListAdapter()).notifyDataSetChanged();
        if (peers.size() == 0) {
            Log.d(WiFiDirectActivity.TAG, "No devices found");
            return;
        }

    }
```
*Los métodos ```public void clearPeers()``` y ```public void onInitiateDiscovery``` estan delegados para limpiar la lista de peers disponibles y para mantener activo el  ```progressDialog``` mientras se esta realizando una búsqueda de peers en la red respectivamente.
```java
public void clearPeers() {
        peers.clear();
        ((WiFiPeerListAdapter) getListAdapter()).notifyDataSetChanged();
    }

    /**
     * 
     */
    public void onInitiateDiscovery() {
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
        progressDialog = ProgressDialog.show(getActivity(), "Press back to cancel", "finding peers", true,
                true, new DialogInterface.OnCancelListener() {

                    @Override
                    public void onCancel(DialogInterface dialog) {
                        
                    }
                });
    }
```
* Por último , se declara la interfaz ```DeviceActionListener``` que será de utilidad para la actividad principal, debido a que permite escuchar eventos asociados a un fragmento especifico del ```ListFragment```.
```java
  public interface DeviceActionListener {

        void showDetails(WifiP2pDevice device);

        void cancelDisconnect();

        void connect(WifiP2pConfig config);

        void disconnect();
    }

}
```
**Creación del WiFiDirectActivity:**<br/>
Ya para concluir, se se crea la actividad principal que en general usa Wi-Fi Direct para descubir, conectar y transferir imagenes con dispositivos disponibles en la red.<br/>
Inicalmente de declaran las variables globales.Después con el método ```public void setIsWifiP2pEnabled(boolean isWifiP2pEnabled)``` se modifica el estado del Wi-FI por medio del ```WiFiDirectBroadcastReceiver``` además en el ```public void onCreate(Bundle savedInstanceState)``` se definen los tipos de intentos asociados a Wi-Fi Direct , que ya se han explicado previamente y se inicializa el objeto ```WifiP2pManager manager``` responsable de la gestión de la conectividad de Wi-Fi Direct.
```java
import android.app.Activity;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.net.wifi.p2p.WifiP2pConfig;
import android.net.wifi.p2p.WifiP2pDevice;
import android.net.wifi.p2p.WifiP2pManager;
import android.net.wifi.p2p.WifiP2pManager.ActionListener;
import android.net.wifi.p2p.WifiP2pManager.Channel;
import android.net.wifi.p2p.WifiP2pManager.ChannelListener;
import android.os.Bundle;
import android.provider.Settings;
import android.util.Log;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.widget.Toast;

import com.example.android.wifidirect.DeviceListFragment.DeviceActionListener;


 
public class WiFiDirectActivity extends Activity implements ChannelListener, DeviceActionListener {

    public static final String TAG = "wifidirectdemo";
    private WifiP2pManager manager;
    private boolean isWifiP2pEnabled = false;
    private boolean retryChannel = false;

    private final IntentFilter intentFilter = new IntentFilter();
    private Channel channel;
    private BroadcastReceiver receiver = null;

    /**
     * @param isWifiP2pEnabled the isWifiP2pEnabled to set
     */
    public void setIsWifiP2pEnabled(boolean isWifiP2pEnabled) {
        this.isWifiP2pEnabled = isWifiP2pEnabled;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        // add necessary intent values to be matched.

        intentFilter.addAction(WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION);
        intentFilter.addAction(WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION);
        intentFilter.addAction(WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION);
        intentFilter.addAction(WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION);

        manager = (WifiP2pManager) getSystemService(Context.WIFI_P2P_SERVICE);
        channel = manager.initialize(this, getMainLooper(), null);
    }
```

* Ahora bien, cuando la actividad esta a punto de ser lanzada al usuario en el ```public void onResume()``` se crea una instancia del ```WiFiDirectBroadcastReceiver```para mantener la comunicación con la actividad principal y además se registra en el manifiesto el BroadcastReceiver con que tipo de ```Intent``` está asociado. <br/>
Sin embargo, cuando la actividad se pausa en el ```public void onPause()``` el BroadcastReceiver se quita del manifiesto.
```java
  @Override
    public void onResume() {
        super.onResume();
        receiver = new WiFiDirectBroadcastReceiver(manager, channel, this);
        registerReceiver(receiver, intentFilter);
    }

    @Override
    public void onPause() {
        super.onPause();
        unregisterReceiver(receiver);
    }
```
* El método ```public void resetData() ``` elimina todos los dispositivos de la lista de peers disponibles y limpia todos los campos, es llamado cuando el BroadcastReceiver recibe un evento asociado a un cambio de estado en la red.
```java
   public void resetData() {
        DeviceListFragment fragmentList = (DeviceListFragment) getFragmentManager()
                .findFragmentById(R.id.frag_list);
        DeviceDetailFragment fragmentDetails = (DeviceDetailFragment) getFragmentManager()
                .findFragmentById(R.id.frag_detail);
        if (fragmentList != null) {
            fragmentList.clearPeers();
        }
        if (fragmentDetails != null) {
            fragmentDetails.resetViews();
        }
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.action_items, menu);
        return true;
    }
  ```
* El método ```public boolean onOptionsItemSelected(MenuItem item) ``` se ejecuta cuando se selecciona una opción del menú:
* Cuando se escoje la opción  **Settings**, se llama a la **configuración de conexiones inalambricas y redes** que tiene por defecto el dispositivo, para que el usuario pueda activar Wi-FI Direct.</br>
* Cuando se escoje la opción **Buscar** se verifica que Wi-FI Direct este activado, en tal caso  se utiliza el objeto ```WifiP2pManager manager``` para encontrar los peers disponibles en dicho momento.
  
```java  
  @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.atn_direct_enable:
                if (manager != null && channel != null) {

                    // Since this is the system wireless settings activity, it's
                    // not going to send us a result. We will be notified by
                    // WiFiDeviceBroadcastReceiver instead.

                    startActivity(new Intent(Settings.ACTION_WIRELESS_SETTINGS));
                } else {
                    Log.e(TAG, "channel or manager is null");
                }
                return true;

            case R.id.atn_direct_discover:
                if (!isWifiP2pEnabled) {
                    Toast.makeText(WiFiDirectActivity.this, R.string.p2p_off_warning,
                            Toast.LENGTH_SHORT).show();
                    return true;
                }
                final DeviceListFragment fragment = (DeviceListFragment) getFragmentManager()
                        .findFragmentById(R.id.frag_list);
                fragment.onInitiateDiscovery();
                manager.discoverPeers(channel, new WifiP2pManager.ActionListener() {

                    @Override
                    public void onSuccess() {
                        Toast.makeText(WiFiDirectActivity.this, "Discovery Initiated",
                                Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onFailure(int reasonCode) {
                        Toast.makeText(WiFiDirectActivity.this, "Discovery Failed : " + reasonCode,
                                Toast.LENGTH_SHORT).show();
                    }
                });
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }
```
* Ahora se pasa a la implementación de los métodos definidos por la interfaz ``` DeviceActionListener ```:<br/>
* El método ```public void showDetails(WifiP2pDevice device)``` muestra la información de un dispositivo en un ```Fragment```.<br/> 
* El método ```public void connect(WifiP2pConfig config)``` llama al objeto ```WifiP2pManager manager``` para empezar una conexión con un dispositivo de acuerdo a una configuración establecida.<br/> 
* El método ```public void disconnect()``` elimina el grupo p2p actual y limpia el ```Fragment``` asociado al dispositivo con el que se estaba conectado.<br/> 
* El método ```public void cancelDisconnect()``` cancela la conexión que apenas se estaba tratando de negociar entre dos dispositivos.
  ```java 
  @Override
    public void showDetails(WifiP2pDevice device) {
        DeviceDetailFragment fragment = (DeviceDetailFragment) getFragmentManager()
                .findFragmentById(R.id.frag_detail);
        fragment.showDetails(device);

    }

    @Override
    public void connect(WifiP2pConfig config) {
        manager.connect(channel, config, new ActionListener() {

            @Override
            public void onSuccess() {
                // WiFiDirectBroadcastReceiver will notify us. Ignore for now.
            }

            @Override
            public void onFailure(int reason) {
                Toast.makeText(WiFiDirectActivity.this, "Connect failed. Retry.",
                        Toast.LENGTH_SHORT).show();
            }
        });
    }

    @Override
    public void disconnect() {
        final DeviceDetailFragment fragment = (DeviceDetailFragment) getFragmentManager()
                .findFragmentById(R.id.frag_detail);
        fragment.resetViews();
        manager.removeGroup(channel, new ActionListener() {

            @Override
            public void onFailure(int reasonCode) {
                Log.d(TAG, "Disconnect failed. Reason :" + reasonCode);

            }

            @Override
            public void onSuccess() {
                fragment.getView().setVisibility(View.GONE);
            }

        });
    }
  
   @Override
    public void cancelDisconnect() {
        if (manager != null) {
            final DeviceListFragment fragment = (DeviceListFragment) getFragmentManager()
                    .findFragmentById(R.id.frag_list);
            if (fragment.getDevice() == null
                    || fragment.getDevice().status == WifiP2pDevice.CONNECTED) {
                disconnect();
            } else if (fragment.getDevice().status == WifiP2pDevice.AVAILABLE
                    || fragment.getDevice().status == WifiP2pDevice.INVITED) {

                manager.cancelConnect(channel, new ActionListener() {

                    @Override
                    public void onSuccess() {
                        Toast.makeText(WiFiDirectActivity.this, "Aborting connection",
                                Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onFailure(int reasonCode) {
                        Toast.makeText(WiFiDirectActivity.this,
                                "Connect abort request failed. Reason Code: " + reasonCode,
                                Toast.LENGTH_SHORT).show();
                    }
                });
            }
        }

    }
```
* Por último se implementa el método ```public void onChannelDisconnected()``` definido por la interfaz  ```ChannelListener``` que se lanza cuando el canal ha sido desconectado y lo que se busca es relanzar de nuevo el canal por medio del método ```initialize(Context, Looper, WifiP2pManager.ChannelListener)```.
```java
    @Override
    public void onChannelDisconnected() {
        // we will try once more
        if (manager != null && !retryChannel) {
            Toast.makeText(this, "Channel lost. Trying again", Toast.LENGTH_LONG).show();
            resetData();
            retryChannel = true;
            manager.initialize(this, getMainLooper(), this);
        } else {
            Toast.makeText(this,
                    "Severe! Channel is probably lost premanently. Try Disable/Re-Enable P2P.",
                    Toast.LENGTH_LONG).show();
        }
    }
}
 ```
 
