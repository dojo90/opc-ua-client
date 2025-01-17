![robot][1]

# opc-ua-client

Communicate using OPC Unified Architecture and Visual Studio. With this library, your app can browse, read, write and subscribe to the live data published by the OPC UA servers on your network.

Supports .NET Core, Universal Windows Platform (UWP), Windows Presentation Framework (WPF) and Xamarin applications.

### Getting Started

Install package 'Workstation.UaClient' from Nuget to get the latest release for your hmi project.

Here's an example of reading the variable "ServerStatus" from a public OPC UA server.

```
using System;
using System.Threading.Tasks;
using Workstation.ServiceModel.Ua;
using Workstation.ServiceModel.Ua.Channels;
					
public class Program
{
    /// <summary>
    /// Connects to server and reads the current ServerState. 
    /// </summary>
    public static async Task Main()
    {
        // describe this client application.
        var clientDescription = new ApplicationDescription
        {
            ApplicationName = "Workstation.UaClient.FeatureTests",
            ApplicationUri = $"urn:{System.Net.Dns.GetHostName()}:Workstation.UaClient.FeatureTests",
            ApplicationType = ApplicationType.Client
        };

        // create a 'UaTcpSessionChannel', a client-side channel that opens a 'session' with the server.
        var channel = new UaTcpSessionChannel(
            clientDescription,
            null, // no x509 certificates
            new AnonymousIdentity(), // no user identity
            "opc.tcp://opcua.rocks:4840", // the public endpoint of a server at opcua.rocks.
            SecurityPolicyUris.None); // no encryption
        try
        {
            // try opening a session and reading a few nodes.
            await channel.OpenAsync();

            Console.WriteLine($"Opened session with endpoint '{channel.RemoteEndpoint.EndpointUrl}'.");
            Console.WriteLine($"SecurityPolicy: '{channel.RemoteEndpoint.SecurityPolicyUri}'.");
            Console.WriteLine($"SecurityMode: '{channel.RemoteEndpoint.SecurityMode}'.");
            Console.WriteLine($"UserIdentityToken: '{channel.UserIdentity}'.");

            // build a ReadRequest. See 'OPC UA Spec Part 4' paragraph 5.10.2
            var readRequest = new ReadRequest {
                // set the NodesToRead to an array of ReadValueIds.
                NodesToRead = new[] {
                    // construct a ReadValueId from a NodeId and AttributeId.
                    new ReadValueId {
                        // you can parse the nodeId from a string.
                        // e.g. NodeId.Parse("ns=2;s=Demo.Static.Scalar.Double")
                        NodeId = NodeId.Parse(VariableIds.Server_ServerStatus),
                        // variable class nodes have a Value attribute.
                        AttributeId = AttributeIds.Value
                    }
                }
            };
            // send the ReadRequest to the server.
            var readResult = await channel.ReadAsync(readRequest);

            // DataValue is a class containing value, timestamps and status code.
            // the 'Results' array returns DataValues, one for every ReadValueId.
            var serverStatus = readResult.Results[0].GetValueOrDefault<ServerStatusDataType>();

            Console.WriteLine("\nServer status:");
            Console.WriteLine("  ProductName: {0}", serverStatus.BuildInfo.ProductName);
            Console.WriteLine("  SoftwareVersion: {0}", serverStatus.BuildInfo.SoftwareVersion);
            Console.WriteLine("  ManufacturerName: {0}", serverStatus.BuildInfo.ManufacturerName);
            Console.WriteLine("  State: {0}", serverStatus.State);
            Console.WriteLine("  CurrentTime: {0}", serverStatus.CurrentTime);

            Console.WriteLine($"\nClosing session '{channel.SessionId}'.");
            await channel.CloseAsync();
        }
        catch(Exception ex)
        {
		 	 await channel.AbortAsync();
            Console.WriteLine(ex.Message);
        }
    }
}

// Server status:
//   ProductName: open62541 OPC UA Server
//   SoftwareVersion: 1.0.0-rc5-52-g04067153-dirty
//   ManufacturerName: open62541
//   State: Running
//   CurrentTime: 09/08/2019 17:54:14

```

### Model, View, ViewModel (MVVM)

For HMI applications, you can use XAML bindings to connect your UI elements to live data.

First add a UaApplication instance and initialize it during startup:
```
public partial class App : Application
{
    private UaApplication application;

    protected override void OnStartup(StartupEventArgs e)
    {
        // Build and run an OPC UA application instance.
        this.application = new UaApplicationBuilder()
            .SetApplicationUri($"urn:{Dns.GetHostName()}:Workstation.StatusHmi")
            .SetDirectoryStore($"{Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData)}\\Workstation.StatusHmi\\pki")
            .SetIdentity(this.ShowSignInDialog)
            .Build();

        this.application.Run();

        // Create and show the main view.
        var view = new MainView();
        view.Show();
    }
	...
}
```

Then any view model can be transformed into a OPC UA subscription.  
```    
[Subscription(endpointUrl: "opc.tcp://localhost:26543", publishingInterval: 500, keepAliveCount: 20)]
public class MainViewModel : SubscriptionBase
{
    /// <summary>
    /// Gets the value of ServerServerStatus.
    /// </summary>
    [MonitoredItem(nodeId: "i=2256")]
    public ServerStatusDataType ServerServerStatus
    {
        get { return this.serverServerStatus; }
        private set { this.SetValue(ref this.serverServerStatus, value); }
    }

    private ServerStatusDataType serverServerStatus;
}
```
[1]: robot6.jpg  
