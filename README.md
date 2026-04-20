# Getting Started with Hands-On Exercises

During this session, we will work with Zowe configuration files to understand their behavior and how they operate with the Zowe CLI.
To access the terminal in this environment, you can use the **Terminal** menu at the top and select **New Terminal**.

To start, ensure the Zowe CLI is installed.

```bash
zowe --help
```

If a list of commands is displayed, the system is ready. Otherwise, Zowe CLI needs to be installed.

## Creating our first configuration file

If there is no `zowe.config.json` file, one needs to be created. The easiest way to create one is by using Zowe. Zowe works with multiple configuration files. To start, there should *almost always* be a global configuration file.

The global configuration file is used when no other files are present. Think of it as the base layer upon which everything else is applied or the configuration file of last resort, assuming it cannot find another.

To get started, the global config file needs to be created. It should be created in the `$HOME/.zowe` folder.

```bash
zowe config init --gc
```

> [!TIP]
> If you are working in an environment with the Zowe API Mediation Layer (API ML), you can also initialize your configuration automatically. See the [Appendix](#zowe-configuration-from-api-ml) for more details.

**NOTE:** This environment is not secure, different topic for another day.  You will get a message like this: 
```
Warning:
Unable to securely save credentials. Skipped prompting for credentials.

Possible Solutions:
 - Reinstall Zowe CLI. On Linux systems, also make sure to install the prerequisites listed in Zowe CLI documentation.
 - Ensure Zowe CLI can access secure credential storage. Zowe CLI needs access to the OS to securely save credentials.

Saved config template to /home/developer/.zowe/zowe.config.json
```


Once created, navigate to the global folder. You can access it using:

```cd ~/.zowe``` on Mac/Linux environments.

List the folder contents using the `ls` command:

```bash
ls -la
```

There will be various files in there. The two files of interest are `zowe.config.json` and `zowe.config.schema`.

### Zowe Configuration JSON

`zowe.config.json` contains all the settings that the Zowe CLI and Zowe Explorer will use during execution. Execute the following from the terminal to open it in the editor:

```bash
code zowe.config.json
```

Look through the `zowe.config.json` file. It contains various values; most will be blank and need to be configured.

For this environment, some settings need adjustment for the profile to connect properly.

#### Start with global_base
`global_base` is a special **base profile** defined within the `profiles` object. Its purpose is to act as a "parent" or "common" configuration for other profiles.

*   **Purpose**: Instead of re-typing your `host`, `port`, `user`, and `password` for every service (zosmf, tso, ssh, etc.), you define them once in `global_base`.
*   **Inheritance**: Other service profiles "inherit" from the base profile. When you run a `zosmf` command, Zowe CLI looks for any missing connection details in the base profile.
*   **Common Properties**:
    *   `host`: The mainframe hostname or IP address.
    *   `port`: The API port.
    *   `user`: Your mainframe ID.
    *   `password`: Your mainframe password.
    *   `rejectUnauthorized`: Controls whether to allow self-signed certificates.

By using `global_base`, you only have to update your password in one place when it expires, rather than in dozens of individual service profiles.

Notice how there's a name, `global_base` and a type of `base`.  Multiple base profiles can exist in `zowe.config.json`.  They can be selected using the Zowe CLI.
Under `base`, a couple of default parameters are listed under `properties`, such as `host` and `rejectUnauthorized`.  

The current version shows:
```json
        "global_base": {
            "type": "base",
            "properties": {
                "host": "",
                "rejectUnauthorized": true
            },
            "secure": [
                "user",
                "password"
            ]
        }
```

Let's change those values and add a couple of more to make it look like this:
```json
        "global_base": {
            "type": "base",
            "properties": {
                "host": "10.1.2.123",
                "rejectUnauthorized": false,
                "user": "$ZOWE_OPT_USER",
                "password": "$ZOWE_OPT_PASSWORD"
            },
            "secure": []
        }
```

We added the host IP address, so Zowe can connect to it.
- `rejectUnauthorized` checks to ensure the certificate by the host is signed.  Setting the flag allows us to interact with self-signed certificates, as in the current use case.
- `user` and `password` have been set up to use environment variables.  This environment does not have all the security systems installed, so we inject the username and password using environment variables.  


Now, let's change the connection to z/OSMF to the proper port.  The default port for z/OSMF is 443, but this system uses 10443.  We can't put it in the `global_base` profile, as each service can have its own port (with Zowe API Mediation Layer all services can be served under a single port).  Find the `zosmf` section (near the top) and change it from this:

```json
        "zosmf": {
            "type": "zosmf",
            "properties": {
                "port": 443
            },
            "secure": []
        },
```

to this:

```json
        "zosmf": {
            "type": "zosmf",
            "properties": {
                "port": 10443
            },
            "secure": []
        },
```

**NOTE:** In standard environments, the `"secure"` section should be filled with any value you don't want to share.  In this case, it is still secure, as we are not exposing the username and password.  


## Inspect Zowe Configuration Schema

Open the `zowe.config.schema` file by typing this command in the terminal:

```bash
code zowe.config.schema
```

**Do not make any modifications to this file.**

Just look through it. This file shows all available options for each configuration property in the `zowe.config.json` file.

This file is automatically updated when new plugins are installed using the Zowe CLI, reflecting the configuration settings available for the new Zowe plugins. It also helps the editor validate the configuration file.

Most users will never need to modify this file, and it is not strictly required. However, it is supported by editors like Visual Studio Code to provide property prompting and autocompletion while editing `zowe.config.json`. If it exists, leave it as is.

Close the `zowe.config.schema` file.

# Working with Parameters

## Our first order of precedence test

Run the Zowe command to list datasets.

```bash
zowe files list ds "cust001.*"
```

What happened?  

It should list a set of files:
```
CUST001.BRIAN.GDG
CUST001.BRIGHT.MARBLES.COBOL
CUST001.BRIGHT.MARBLES.JCL
...
CUST001.WORKSHOP.TEST4Z.JCL
CUST001.WORKSHOP.TEST4Z.LOADLIB
```

Now, run a command with an option to override a value. The `zowe.config.json` file stores the username and password, either directly in the file or in a secure credential vault.

Let's cause the command to fail by providing an incorrect password:

```bash
zowe files list ds "cust001.*" --password wrong
```

This overrides the value in the `zowe.config.json` file located in `~/.zowe`.

The command should fail with a message about an incorrect username and/or password.

What happened?  You got an error: 
```
Unable to perform this operation due to the following problem.
This operation requires authentication.
Username or password are not valid or expired.

Response From Service
Error: Rest API failure with HTTP(S) status 401

Diagnostic Information
Received HTTP(S) error 401 = Unauthorized.

Protocol:          https
Host:              10.1.2.123
Port:              10443
Base Path:         
Resource:          /zosmf/restfiles/ds?dslevel=cust001.*
Request:           GET
Headers:           [{"Accept-Encoding":"gzip"},{"X-IBM-Max-Items":"0"},{"X-CSRF-ZOSMF-HEADER":true}]
Payload:           undefined
Allow Unauth Cert: true
Available creds:   user,password,base64EncodedAuth
Your auth order:   basic,token,bearer,cert-pem
Auth type used:    basic
```

An important aspect of learning Zowe CLI is reading error messages.  They will often contain an HTTP(S) error code.  If you look up HTTP 401 ERROR, you will get a message about being unauthorized.  That's the same thing the detailed message shows.

Try this, override the port value to the default, 443, by running:

```bash
zowe files list ds "cust001.*" --port 443
```

Look at the error message:
```
Unable to perform this operation due to the following problem.
Failed to send an HTTP request.

Response From Service
Error: connect ECONNREFUSED 10.1.2.123:443

Diagnostic Information
HTTP(S) client encountered an error. Request could not be initiated to host.
Review connection details (host, port) and ensure correctness.

Protocol:          https
Host:              10.1.2.123
Port:              443
Base Path:         
Resource:          /zosmf/restfiles/ds?dslevel=cust001.*
Request:           GET
Headers:           [{"Accept-Encoding":"gzip"},{"X-IBM-Max-Items":"0"},{"X-CSRF-ZOSMF-HEADER":true}]
Payload:           undefined
Allow Unauth Cert: true
Available creds:   user,password,base64EncodedAuth
Your auth order:   basic,token,bearer,cert-pem
Auth type used:    basic
```

Zowe CLI returns a different error type, `ECONNREFUSED`.   This indicates Zowe was unable to connect to the service.  In this case, it reached the system, but the system refused the connection.  

As an exercise, you can try other commands that will give an error using parameters like `--rejectUnauthorized true` or  `--protocol http`.  Each one gives a different error and explains the problem. Zowe will always return after an error, but if there's more than one error, it will stop at the first one.  If you have multiple errors, you will correct each one until it works.

## Our second order of precedence test

Running the basic Zowe command again should display the files:

```bash
zowe files list ds "cust001.*"
```

Now, let's create an environment variable to override a value:

```bash
export ZOWE_OPT_REJECT_UNAUTHORIZED=true
zowe files list ds "cust001.*"
```

Did an error message appear? If so, the environment variable overrode the reject unauthorized value in the `zowe.config.json` file.

### Resetting the flag

Zowe will continue to use the environment variable until it is unset. This can be done by opening a new terminal, closing the current one, or manually unsetting the variable.

**NOTE: In a Linux/Mac environment unset the variable with:**

```bash
unset ZOWE_OPT_REJECT_UNAUTHORIZED
zowe files list ds "cust001.*"
```

*NOTE:* Many variables can be overridden this way, including username, password, hostnames, etc.

# More on Configuration Files

- A **user configuration file** stores user profiles and is used by individuals who need their own unique properties to run commands.
- A **team configuration file** stores team profiles and is shared by a group of people who need the same properties to run commands.
- A **global configuration file** resides in the `ZOWE_CLI_HOME` directory (typically `~/.zowe`). It contains global user and team profiles.
- A **project configuration file** resides in a directory of your choice. It contains project-specific user and team profiles.

## Working with multiple configuration files

We are working with a limited environment, so these configuration files are going to be organized into working and non-working.  But this will illustrate the principles all the same. 

Now, let's work with multiple configuration files. First, see the current behavior using the Zowe command:

```bash
zowe files list ds cust001
```

It should list a set of elements.

We can start by using a configuration file based on the basic global configuration.

Create a copy of the global file using Zowe:

```bash
mkdir not-working
cd not-working
zowe config import ~/.zowe/zowe.config.json
```


This creates a new `zowe.config.json` file in the current directory.  Now verify the file was created by running
`ls -al`

You can edit it by opening it in your editor via the file navigator or by running:

```bash
code zowe.config.json
```

### Changing a value

In the editor, search for `"rejectUnauthorized": false,` and replace it `"rejectUnauthorized": true`,

Save the file and run:

```bash
zowe files list ds cust001
```

You should have received this error:
```bash
Unable to perform this operation due to the following problem.
Failed to send an HTTP request.

Response From Service
Error: certificate has expired

Diagnostic Information
HTTP(S) client encountered an error. Request could not be initiated to host.
Review connection details (host, port) and ensure correctness.

Protocol:          https
Host:              10.1.2.123
Port:              10443
Base Path:         
Resource:          /zosmf/restfiles/ds?dslevel=cust001
Request:           GET
Headers:           [{"Accept-Encoding":"gzip"},{"X-IBM-Max-Items":"0"},{"X-CSRF-ZOSMF-HEADER":true}]
Payload:           undefined
Allow Unauth Cert: false
Available creds:   user,password,base64EncodedAuth
Your auth order:   basic,token,bearer,cert-pem
Auth type used:    basic
```

It's now verifying the certificate and it reports that it is expired.

Now, run the following commands:

```bash
cd ..
zowe files list ds cust001
```

What happened? It should work again.

*Why did that happen?*

Zowe used the local configuration file to override the values in the global configuration. If you `cd` back into the `not-working` directory, it will return to showing the error again.

Knowing that configurations can be stored in different directories, you can have scripts, commands or data specific to one LPAR and change to another folder and have it connect to the corresponding LPAR.  This simplifies access as you don't have to override values.  Multiple configuration files help solve for that.

## Creating a user configuration

User configuration files are used for specific information, typically unique to a given user. These are less common and primarily used for user-specific settings.

As configuration files are aggregated, the `zowe.config.user.json` file will overlay information, including in the current directory.

```bash
zowe config import --uc ~/.zowe/zowe.config.json
ls -la
code zowe.config.user.json
```

This should open the `zowe.config.user.json` file from the current directory.

If you change a value, you will see it even overrides the value in the local `zowe.config.json` file.  In our case, the local `zowe.config.json` file has `"rejectUnauthorized": true` rejecting our connection.  The global file was copied into `zowe.config.user.json` which has the flag set to `false`.  It will override the `zowe.config.json` file, allowing the command to run again.

To see this in action, let's delete the `zowe.config.user.json` file and re-run the dataset list command. Without the user configuration overriding the local `not-working/zowe.config.json` file, the connection should fail again due to the certificate error.

```bash
rm zowe.config.user.json
zowe files list ds cust001
```

It should fail again.

*Tip:* Removing unneeded sections will make management easier.

The format in `zowe.config.user.json` is exactly the same as `zowe.config.json`, but it does not have to have all the same information.  This is a complete, functional user config file.  If there are matches in the next configuration file (up the directory structure), it will override the value.  Otherwise, it will add that to the profile.  In this case, we've reduced the file to just the zosmf profile, keeping it lean, changed the port (to an invalid port, in this case), and added the `"rejectUnauthorized": false` flag.
```json
{
    "$schema": "./zowe.schema.json",
    "profiles": {
        "zosmf": {
            "type": "zosmf",
            "properties": {
                "port": 10444,
                "rejectUnauthorized": false
            },
            "secure": []
        }
    }
}
```

Just to prove this, if you copy that as the only text into `zowe.config.user.json`, the call will fail, indicating a bad connection. 

When using these `zowe.config.user.json` snippet files, they are not complete.  While they can be shared, they can not be used as the sole connection information.  They are only overriding values, and you'd need a more comprehensive version with all the connection details to work.  For example, this is missing user, password, hostname, etc.  


# Configuration File Updates

When a new plugin is installed, `zowe.config.schema` gets updated. This updates all schema files with new information, aiding tools like Visual Studio Code.

Let's see how the schema affects the editor. In the terminal:

```bash
zowe plugins install @zowe/cics-for-zowe-cli
zowe config update-schemas
```

This will update the schema, including files up the directory structure.  You can control how many levels are updated, if you need to control for that.  The update schemas command will show the schema files updated.

```bash
code zowe.config.json
```

Let's edit the file.  Now, when editing the file, the editor will guide you through creating the settings. The TAB key will move from field to field. The editor now has the schema information, which assists with linting and ensuring settings are in the proper format.

Move the cursor to line 10 (It should be `},` closing the `zosmf` section). Hit ENTER to create a new line.
Start typing:
```json
        "cics" : {
            "type": "cics",
```
You will notice it offers "cics" as a type ahead for you.

Now, add more to the profile by continuing:
```json
            "properties": {
                "cicsPlex": ""
            }
```
As you put the " for `cicsPlex`, it will offer that as a suggestion.

To complete this snippet, it should look something like this, and don't forget the last comma (,):
```json
        "cics" : {
            "type": "cics",
            "properties": {
                "cicsPlex": "PLEX1",
                "regionName": "CICSTRN1"
            }
        },
```

The schema file makes editing the file much easier.  And if you make a syntax mistake, the editor will highlight it for you.


# Understanding Order of Precedence

Zowe uses multiple configuration files. Often they are in different directories, but sometimes they are in the same directory (like `zowe.config.user.json`). If things aren't working as expected, it is helpful to understand what Zowe is referencing.

Let's inspect the settings from various locations to see how Zowe interprets the values.


```bash
cd ~/not-working
zowe config profiles --show-inputs-only
```

This output shows which configuration files are being used from that location:
```bash
commandValues: 
  show-inputs-only: true
authenticationType: none
authTypeOrder:      basic,token,bearer,cert-pem
locations: 
  - /home/developer/zowe-api-python/not-working/zowe.config.user.json
  - /home/developer/zowe-api-python/not-working/zowe.config.json
  - /home/developer/.zowe/zowe.config.json
```


If you need to inspect the variables, use the following command:

```bash
zowe config list
```

The output shows the values used by the current path:
```bash
profiles: 
  zosmf: 
    type:       zosmf
    properties: 
      port:               10443
      rejectUnauthorized: false
    secure: 
      (empty array)
  cics: 
    type:       cics
    properties: 
      cicsPlex:   PLEX1
      regionName: CICSTRN1
  tso: 
    type:       tso
    properties: 
      account:        
      codePage:       1047
      logonProcedure: IZUFPROC
    secure: 
      (empty array)
  ssh: 
    type:       ssh
    properties: 
      port: 22
    secure: 
      (empty array)
  global_base: 
    type:       base
    properties: 
      host:               10.1.2.123
      rejectUnauthorized: true
      user:               CUST045
      password:           CUST045
    secure: 
      (empty array)
defaults: 
  zosmf: zosmf
  tso:   tso
  ssh:   ssh
  base:  global_base
autoStore: true
```

To determine where those values are set, use the `--locations` flag:

```bash
zowe config list --locations
```

This lists the values for each `zowe.config.json` being used in the current directory and upstream.  The output is much longer, so here's just a snippet:
```bash
/home/developer/zowe-api-python/not-working/zowe.config.user.json: 
  $schema:  ./zowe.schema.json
  profiles: 
    zosmf: 
      type:       zosmf
      properties: 
        port:               10443
        rejectUnauthorized: false
      secure: 
        (empty array)
  defaults: 
/home/developer/zowe-api-python/not-working/zowe.config.json: 
  $schema:   ./zowe.schema.json
  ...
```


# Digging Deeper

Sometimes problems are harder to find. In such cases, there is a command to display all options used:

```bash
zowe config report-env
```

This command includes information about Zowe configuration files and reports versions of tools, home locations, platforms, and `ZOWE_OPT_` environment variables.

Run the command and inspect the output. 


Set an environment variable to see if it is picked up:

```bash
export ZOWE_OPT_NAME=jim
zowe config report-env
```

Did `ZOWE_OPT_NAME` appear in the output? This command is extremely useful for diagnosing issues.

Here's some of the output: 
```bash
______________________________________________
Zowe CLI version = 8.31.4
Node.js version = 20.15.1
O.S. platform = linux
O.S. architecture = x64

O.S. PATH = /home/developer/zowe-api-python/node_modules/.bin:/home/developer/zowe-api-python/not-working/node_modules/.bin:/home/developer/zowe-api-python/node_modules/.bin:/home/developer/node_modules/.bin:/home/node_modules/.bin:/node_modules/.bin:/usr/lib/node_modules/npm/node_modules/@npmcli/run-script/lib/node-gyp-bin:/home/developer/.local/bin:/strong/strong-ide/bin/remote-cli:/home/developer/.sdkman/candidates/grails/current/bin:/home/developer/.local/bin:/strong/bin:/usr/local/go/bin:/home/developer/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/developer/strong-ide/data/User/globalStorage/github.copilot-chat/debugCommand

ZOWE_CLI_HOME = undefined
    Default = /home/developer/.zowe
ZOWE_APP_LOG_LEVEL = undefined
ZOWE_IMPERATIVE_LOG_LEVEL = undefined
ZOWE_OPT_PASSWORD = ******
ZOWE_OPT_USER = CUST045
ZOWE_OPT_NAME = jim


______________________________________________
NPM information:
    
    NPM version = 10.8.2
    Shell = /bin/bash
    Global prefix = /home/developer/.npm-local
        The directory above contains the Zowe Node.js command script.
    Global root node modules = /home/developer/.npm-local/lib/node_modules
    Global config = /home/developer/.npm-local/etc/npmrc
    Local prefix = /home/developer/zowe-api-python
    Local root node modules = /home/developer/zowe-api-python/node_modules
    User config = /home/developer/.npmrc
    
    node bin location = /usr/bin/node
    cwd = /home/developer/zowe-api-python/not-working
    HOME = /home/developer


______________________________________________
Zowe CLI configuration information:

    Zowe daemon mode = off
    Zowe client config files in use:
        /home/developer/zowe-api-python/not-working/zowe.config.user.json
        /home/developer/zowe-api-python/not-working/zowe.config.json
        /home/developer/.zowe/zowe.config.json

```

# Working with Zowe Explorer

Zowe Explorer also works with multiple configuration files. Like the Zowe CLI, it can access the global configuration file or one in the current working directory.

Within the environment, find the Zowe Explorer icon on the left side.  Click it.  It should display a panel showing "Datasets", "Unix System Services" and "Jobs".

Right-click on the profile name, in this case `zosmf`, to open the context menu. Clicking "Manage Profile" will open a dialog at the top of the screen to select various options, including "Update Credentials" and "Edit Profile".

You can open the profile used by Zowe Explorer from this menu.  Zowe Explorer honors the same structures as Zowe CLI.  The `zowe.config.json` files are the exact same, however which one is used depends on the workspace.  In our online environment, it will be in the current project folder.  If a `zowe.config.json` file exists in the project folder, it will open that from the "Edit Profile" menu.  If one doesn't exist, it will open the global version ($HOME/.zowe/zowe.config.json).

To test this, if there's a `zowe.config.json` in your workspace folder, use the Zowe Explorer to "Edit Profile".

It will open the one in the current folder.  How can you tell?  Move the cursor over the tab. It will show something similar to ~/zowe-api-python/zowe.config.json.

Now close the editor.  Delete the `zowe.config.json` file in the main folder.  To do this, you'll need to go to the file browser (upper left icon) and right click, then delete.

Go to Zowe Explorer, right click "zosmf" and Edit Profile again.  Move the cursor over the filename and it will show ~/.zowe/zowe.config.json.

What does that mean?  ~ is a shortcut for the home directory on Linux.  It's referencing the global `zowe.config.json` file. 

On your local system, it will often ask if you want the project folder or the global, if there's a local version of the `zowe.config.json` file.

# The End
Hopefully, you've learned something new about managing the configuration files themselves.  

Congrats on completing the workshop!

# Appendix

## Zowe Configuration from API ML

> [!NOTE]
> This section is for informational purposes as the API Mediation Layer (API ML) is not available in this specific workshop environment.

In environments with API ML, you can initialize your settings automatically by pulling them from the server. This can be used to create global and user configurations with the correct connection settings pre-filled.

To see what would be created, you can use **Dry-Run** mode:

**Global Initialization**:
```bash
zowe config auto-init --gc --dry-run
```

**User Initialization**:
```bash
zowe config auto-init --uc --dry-run
```

What do you notice in the generated output? This process eliminates the need to manually enter hostnames, ports, and other connection properties.
