# Constellation Scripts

A collection of Python Scripts and Jupyter Notebooks.

[Constellation](https://github.com/constellation-app/constellation) is a 
graph-focused data visualisation and interactive analysis application enabling 
data access, federation and manipulation capabilities across large and complex 
data sets.

## What's In This Repository

There are 2 folders being `PythonScripts` and `JupyterNotebooks` that respectively hold scripts and notebooks.

## Accessing Constellation remotely

Constellation has an internal web server which it uses to offer a REST API. By design, this webserver only listens on localhost, so only REST clients running on the same system can reach the server. The Python `Constellation_client` module provided by Constellation by default uses HTTP to communicate with the REST server.

(The transport is HTTP to avoid the complex configuration of HTTPS. Since communication is from `localhost` to `localhost`, there is no network traffic to eavesdrop on.)

To run a Jupyter notebook on a remote system but still use the `Constellation_client` module to communicate with Constellation, some work must be done.

When it starts up, the REST server creates two files.

- `$HOME/.ipython/Constellation_client.py` - this is a Python module that provides a convenient interface to the REST API. In general, it does not change from day to day.
- `$HOME/.CONSTELLATION/rest.json` - this file contains a secret that the client must use when communicating with the server. This stops another client on the same system from hijacking the server.

The secret is unique to each running instance of the REST server. This file also contains the port that the server is listening on.

These files must be present in the corresponding directories on the remote system. In fact, the `Constellation_client.py` file can be anywhere on the Python path that it can be imported. To see the default Python path, start Python and run `import sys;print(sys.path)`.

### Using ssh

An SSH client (such as PuTTY or PLINK on a Windows system, or ssh on a UNIX system) can be used to provide access to the local REST server from a remote system. Although the transport is unencrypted HTTP, the SSH tunnel encrypts the traffic, making the network connection secure. However, the secret is still required so the port can't be hijacked.

#### Configuring ssh

Determine which port the REST server is listening on. This can be found in Constellation by going to Setup -> Options and look in the Constellation tab for Internal Webserver, Listen port. By default this is port 1517. This number will be used below; if you use a different port number, use that instead.

Configure the ssh server on the remote system to allow remote TCP forwarding. On a Linux system as root, edit `/etc/ssh/sshd_config` and set `AllowTcpForwarding` to `remote` or `yes`, then tell `sshd` to reread its configuration file: `kill -HUP $(cat /var/run/sshd.pid)`.

Configure the ssh client on the local system to listen remotely on port 1517 and forward connections to localhost:1517.

Using PuTTY, this is done in the configuration dialog box at SSH -> Tunnels; source port 1517, destination localhost:1517, select Remote, click Add. This results in "R1517 localhost:1517" being added to the list of forwarded ports.

Using ssh (untested):
`ssh -R 1517:localhost:1517 user@remote`

#### Running a notebook

Start Constellation. Create a new graph and add a couple of nodes and a transaction.

Start the REST server (Tools -> Start REST Server) to write the secret file.

Copy the `Constellation_client.py` and `rest.json` files (see above) to their corresponding directories on the remote system. (Use the software of your choice, such as scp or WinSCP.)

Use the ssh client to connect to the remote system. The tunnel should now be active.

On the remote system, start a Jupyter notebook or console. Run the following code.

```python
import pandas as pd
import Constellation_client
cc = Constellation_client.Constellation()
df = cc.get_dataframe()
df.head()
```

To ensure two-way communication:

```python
df = pd.DataFrame({'source.Identifier':['mal@firefly.com']})
cc.put_dataframe(df)
```

This should work nicely.

### Using a shared filesystem

This requires a shared filesystem writeable by both the Constellation system and the remote system running the Jupyter notebook. Using a shared filesystem will be a bit slower than using HTTP, since each end polls the filesystem looking for a file. The polling cannot be too fast, otherwise it will overload the filesystem.

It is assumed that the shared filesystem is secure, so there is no need for a separate secret file.

We assume that Constellation is running on Windows, with the shared fileystem mounted on `I:`, and the Jupyter notebook is running on a remote UNIX system with the shared filesystem mounted on `/home/user`.

#### Configuring Constellation for a shared filesystem

Start Constellation. Create a new graph and add a couple of nodes and a transaction.

Copy the `Constellation_client.py` file (see above) to the corresponding directory on the remote system. (Use scp or WinSCP, or the software of your choice.) (If the `Constellation_client.py` file does not exist, start the REST server to create it. The REST server is otherwise not required here.)

Tell Constellation which directory to use for remote communications. Go to Setup -> Options and look in the Constellation tab for Internal Webserver, REST directory. We will use `I:/.Constellation/REST`. (The `.Constellation` name is used by convention - any name can be used.)

Start the file listener: Tools -> Start/Stop File Listener. If the REST directory does not exist, it will be created.(Note: there is no need to start the REST server.)

The `Constellation_client.Constellation` class uses the `http` transport by default. To use the shared filesystem transport, specify the REST directoy as shown in the code below below. The REST directory here corresponds to the shared directory configured in Constellation.

On the remote system, start a Jupyter notebook or console. Run the following code.

```python
import pandas as pd
import Constellation_client
cc = Constellation_client.Constellation(transport='file=/home/user/.Constellation/REST')
df = cc.get_dataframe()
df.head()
```

The `Constellation` class will display

`Transport: file in /home/user/.Constellation/REST`

on stderr to notify the user that the shared filesystem transport is being used.

To ensure two-way communication:

```python
df = pd.DataFrame({'source.Identifier':['mal@firefly.com']})
cc.put_dataframe(df)
```

Alternatively, to avoid having to edit the notebook when switching between the HTTP transport and the shared filesystem transport, the environment variable `Constellation_TRANSPORT` can be set to specify the transport before starting the Jupyter notebook server. The `Constellation()` class discovers the transport as follows.

```python
ENV_VAR = 'Constellation_TRANSPORT'

class Constellation:
    def __init__(self, transport=None):
        if transport is None:
            env = os.getenv(ENV_VAR)
            if env:
                transport = env
        ...
```

Define the environment variable on Linux as:

```bash
$ export Constellation_TRANSPORT='file=/home/user/.Constellation/REST'
```

The Jupyter notebook can then use

```python
cc = Constellation_client.Constellation()
```

and still use the file transport with the specified shared directory.


## More Information
This repository should follow everything mentioned in the Constellation 
[README](https://github.com/constellation-app/constellation/blob/master/README.md).
