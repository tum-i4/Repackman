# Repackman

Repackaging is a technique adopted by attackers to generate fake, malicious versions of legitimate Android apps, which undermines users' trust in the Android ecosystem. Unfortunately, the process of devising anti-repackaging techniques to detect and prevent repackaging is hindered by the lack of repackaged versions of legitimate apps. We implemented Repackman, a tool to automatically repackage Android
apps with arbitrary payloads.

Given the ability of Repackman to generate malicious versions of legitimate apps, we furnish the source code of the tool only upon request from legitimate, verifiable industry/academia source. Please write us an email if you wish to access the source code of Repackman from your business or academic email address.

## Dependencies
The current implementation depends on the following tools:
* [Apktool](ibotpeaches.github.io/Apktool/): Apktool helps is used by Repackman to disassemble input APK archives and retrieve the apps' classes in the `smali` format. It is also utilized to re-assemble the modified app.
* [androguard](https://github.com/androguard/androguard): Used by Repackman to statically analyze an app and retrieve its components. Our tool uses this information to find candidate locations to inject arbitrary payloads into the app's original `smali` code.
* [sqlite3](https://docs.python.org/2/library/sqlite3.html): Python DB-API 2.0 interface for SQLite databases

## Repackaging Process
Repackman adopts the following process (summarized in figure 1) to graft Android apps with arbitrary payloads:

1. Given the path to the APK archive to repackage, the first step in the repackaging process is to extract the archive and disassemble its classes.dex file using Apktool. This step also includes analyzing the app to retrieve details about its components and their internal structures.
2. We use this information in the second step to identify possible locations to inject the rider code. Such insertion locations are dictated by the user via a command-line argument called `--deploymentmethod`. Users can choose the functionality of the rider code (i.e., payloads), and whether it should be executed only upon the realization of certain conditions (i.e., triggers) in this step.
3. In step three, the payloads and triggers chosen by the user are retrieved from a database which contains `smali` templates of those code segments. The tool supports managing those payloads and triggers including adding new ones to its database.
4. Some of the payloads and triggers need access to specific Android permissions. For example, in order to use the **sendSMS** payload, the repackaged app needs to declare the `android.permission.SEND_SMS` permission. The tool adds the necessary permissions associated with each payload/trigger to the target apps *AndroidManifest.xml* file.
5. In step five, the trigger and payload `smali` are injected into the original app’s code, and necessary changes are made to the mixed code. Lastly, the modified code of the app will be re-assembled using `Apktool` and signed using a private key supplied by the user.

## User Manual

We implemented Repackman in python. The tool supports different modes of operations to manage the payload/trigger templates and repackage apps. The operations currently supported by Repackman are (accessible by running ```python repackman.py```):

* addtemplate
* deletetemplate
* listtemplates
* repack

Each operation has its own command-line arguments. To view such arguments, use the command ```python repackman.py [operation] --help```.

### Adding Templates

This operation adds new text files containing `smali` representations of payloads and/or triggers to Repackman's database. 

```
usage: repackman.py addtemplate [-h] [-p PATH] [-n NAME] [-d DESCRIPTION]
                                [-t {payload,trigger}] [-e DEPLOYMENTMETHODS]
                                [-m PERMISSIONS [PERMISSIONS ...]]
                                [-a DEPENDENCIES [DEPENDENCIES ...]]

optional arguments:
  -h, --help            show this help message and exit
  -p PATH, --path PATH  path to template
  -n NAME, --name NAME  name of template in database
  -d DESCRIPTION, --description DESCRIPTION
                        description of template
  -t {payload,trigger}, --type {payload,trigger}
                        type of template to be added (payload or trigger)
  -e DEPLOYMENTMETHODS, --deploymentmethods DEPLOYMENTMETHODS
                        valid deployment methods for template (a for activity,
                        r for receiver, s for service asr for all
  -m PERMISSIONS [PERMISSIONS ...], --permissions PERMISSIONS [PERMISSIONS ...]
                        needed permissions
  -a DEPENDENCIES [DEPENDENCIES ...], --dependencies DEPENDENCIES [DEPENDENCIES ...]
                        paths and names (name of dependency as used in
                        payload/trigger) to needed dependencies
                        (path:name:type)
```

An example of adding a new template to Repackman is:

```
python repackman.py addtemplate -p "./templates/sendGPSlocation.smali" -n "sends GPS location via SMS" -d "sends GPS location via SMS and displays toast (does not send SMS)" -t payload -e as -m "android.permission.ACCESS\_FINE\_LOCATION" "android.permission.SEND\_SMS" -a "./templates/Location.smali:Location:file"
```

Add a payload template with the name "sends GPS location via SMS", the description "sends GPS location via SMS and displays toast (does not send SMS)", located at "./templates/sendGPSlocation.smali" that can be injected into an activity or service. The payload uses the permissions "android.permission.ACCESS_FINE_LOCATION" and "android.permission.SEND_SMS" and has a file-dependency with the original
name "Location.smali" at the location "./templates/Location.smali:Location:file"

### Deleting Templates

Used to remove a template from the database.

```
usage: repackman.py deletetemplate [-h] [-i ID]

optional arguments:
  -h, --help      show this help message and exit
  -i ID, --id ID  id of template that is to be deleted
```

An example of deleting an existing template from the Repackman database is:

```
python repackman.py deletetemplate -i 4
```

Deleting a template (payload or trigger) from the database with the ID number 4.

### Listing Templates

Lists all `smali` templates currently stored in the tool's database.

```
usage: repackman.py listtemplates [-h] [-t {payload,trigger}]

optional arguments:
  -h, --help            show this help message and exit
  -t {payload,trigger}, --type {payload,trigger}
                        type of template to be listed (payload or trigger)
```

Example to list all payloads:

```
python repackman.py listtemplates -t payload
```

To list all triggers in the database:
```
python repackman.py listtemplates -t payload
```

### Repackaging Apps

The main operation supported by Repackman (i.e., repackaging apps with arbitrary payloads)

```
usage: repackman.py repack [-h] [-a APK] [-i ID]
                           [-d {activity,service,receiver,random}]
                           [-t TRIGGER] [-v TRIGGERVAL] [-r {True,False}]
                           [-k KEY] [-m {True,False}] [-o OUTPUT]
                           [-p PASSPHRASE]

optional arguments:
  -h, --help            show this help message and exit
  -a APK, --apk APK     path to apk
  -i ID, --id ID        id of payload
  -d {activity,service,receiver,random}, --deploymentmethod {activity,service,receiver,random}
                        What type of component to deploy payload in. If random
                        is selected, a random, possible payload is selected as
                        well.
  -t TRIGGER, --trigger TRIGGER
                        trigger ID
  -v TRIGGERVAL, --triggerval TRIGGERVAL
                        value of trigger
  -r {True,False}, --randomize {True,False}
                        set to false to insert payload or payload call into
                        beginning of main class, set to true to insert into
                        random activity, method and line
  -k KEY, --key KEY     path to key
  -m {True,False}, --forcemethod {True,False}
                        force payload or payload call into separate method
  -o OUTPUT, --output OUTPUT
                        path of repackaged apk, default:
                        ./out/[package_name]_repackaged.apk
  -p PASSPHRASE, --passphrase PASSPHRASE
                        passphrase to sign apk
```


Example of repackaging an app:

```
python repackman.py repack -a "~/com.example.myapp.apk" -i 3 -d "service" -t 7 -v "2018-02-20" -r False -m True
```

Repackage the app "~/com.example.myapp.apk" with the payload that has the ID 3 and the trigger that has the ID 7 using the value "2018-02-20". The payload is to be injected into a service, should not be injected into a random location, but into a seperate method. We want to use the default key and output path ("./out/com.example.myapp_repackaged.apk").

## Citation and Contact

For more information about the design and implementation of the tool, please refer to the paper cited below. Kindly consider citing our paper, if you find it useful in your research.

```
Coming Soon
```

We are constantly updating the source code and its corresponding documentation. However, should you have any inquiries about installing and using the code, please contact us:

Alei Salem (salem@in.tum.de) and Franziska Paulus (paulusf@in.tum.de)
