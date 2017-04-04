# Uptane
Reference Implementation and demonstration code for [UPTANE](https://uptane.org).

Please note that extensive documentation on design can be found in the following documents:

* [Design Overview](https://docs.google.com/document/d/1pBK--40BCg_ofww4GES0weYFB6tZRedAjUy6PJ4Rgzk/edit?usp=sharing)
* [Implementation Specification](https://docs.google.com/document/d/1wjg3hl0iDLNh7jIRaHl3IXhwm0ssOtDje5NemyTBcaw/edit?usp=sharing)
* [Deployment Considerations](https://docs.google.com/document/d/17wOs-T7mugwte5_Dt-KLGMsp-3_yAARejpFmrAMefSE/edit?usp=sharing)

# Instructions on use of the Uptane demonstration code
## Installation
(As usual, virtual environments are recommended for development and testing, but not necessary. If you use a virtual environment, use python2: virtualenv -p python2 <name>)

Some development libraries are necessary to install some of Uptane's dependencies. If your system uses apt, the command to install them will be:
```shell
sudo apt-get install build-essential libssl-dev libffi-dev python-dev python3-dev
```


To download and install the Uptane code and its dependencies, run the following:
```shell
git clone https://github.com/uptane/uptane
cd uptane
pip install -r dev-requirements.txt
```

Note that the demonstration now operates using ASN.1 / DER format and encoding for metadata files by default. The TUF branch in use has been switched accordingly (so please run the command above again if you have an existing installation). This can be switched back to JSON (which is human readable) by changing the tuf.conf.METADATA_FORMAT option in uptane/\_\_init\_\_.py.


## Running the demo
The code below is intended to be run in five or more consoles:
- WINDOW 1: Python shell for the Image Repository. This serves HTTP (repository files, including metadata).
- WINDOW 2: Python shell for the Director (Repository and Service). This serves metadata and image files via HTTP,1
 and receives manifests from the Primary via XMLRPC.
- WINDOW 3: Bash shell for the Timeserver. This serves signed times in response to requests from the Primary via XMLRPC.
- WINDOW 4: Python shell for a Primary client in the vehicle. This fetches images and metadata from the repositories via HTTP, and communicates with the Director service, Timeserver, and any Secondaries via XMLRPC. (More of these can be run, simulating more vehicles with one Primary each.)
- WINDOW 5: Python shell for a Secondary in the vehicle. This communicates directly only with the Primary via XMLRPC, and will perform full metadata verification. (More of these can be run, simulating more ECUs in one or more vehicles.)


### *WINDOW 1: the Image Repository*
These instructions start a demonstration version of an OEM's or Supplier's main repository
for software, hosting images and the metadata Uptane requires.

```Bash
$ python
Python 2.7.6 (default, Oct 26 2016, 20:30:19) 
[GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

```python
>>> import demo.demo_oem_repo as do
>>> do.clean_slate()
```
After the demo, to end hosting:
```python
>>> do.kill_server()
```


### *WINDOW 2: the Director*
The following starts a Director server, which generates metadata for specific
vehicles indicating which ECUs should install what firmware (validated against
and obtained from the OEM's main repository). It also receives and validates
Vehicle Manifests from Primaries, and the ECU Manifests from Secondaries
within the Vehicle Manifests, which capture trustworthy information about what
software is running on the ECUs, along with signed reports of any attacks
observed by those ECUs.

```python
>>> import demo.demo_director as dd
>>> dd.clean_slate()
```

After that, proceed to the following Windows to prepare clients.
Once those are ready, you can perform a variety of modifications and attacks.
Various manipulations can be made here to the Director's interface. Examples
will be discussed below in the [Delivering an Update](#delivering-an-update)
and [Blocking Attacks](#blocking-attacks) sections.

To end HTTP hosting of the image and director repositories, kill_server()
should be called, otherwise you'll have a zombie Python process.
Killing the HTTP server does not terminate the XMLRPC server, which requires
exiting the shell.

```python
>>> dd.kill_server()
```



### *WINDOW 3: the Timeserver:*
The following starts a simple Timeserver, which receives requests for signed
times, bundled by the Primary, and produces a signed attestation that includes
the nonces each Secondary ECU sent the Primary to include along with the
time request, so that each ECU can better establish that it is not being tricked
into accepting a false time.
```Bash
#!/bin/bash
python demo/demo_timeserver.py
```

### *WINDOW 4(+): the Primary client(s):*
(Image Repo, Director, and Timeserver must already have finished starting up.)
The Primary client started below is likely to run on a more capable and
connected ECU in the vehicle - potentially the head unit / infotainment. It will
obtain metadata and images from the OEM Repository as instructed by the Director
and distribute them appropriately to other, Secondary ECUs in the vehicle,
and it will receive ECU Manifests indicating the software on each Secondary ECU,
and bundle these into a Vehicle Manifest which it will send to the Director.
```python
>>> import demo.demo_primary as dp
>>> dp.clean_slate() # sets up a fresh Primary that has never been updated
>>> dp.update_cycle()
```

The Primary's update_cycle() call:
- fetches and validates all signed metadata for the vehicle, from the Director and Image repositories
- fetches all images that the Director instructs this vehicle to install, excluding any that do not exactly match corresponding images on the Image repository. Any images fetched from the repositories that do not match validated metadata are discarded.
- queries the Timeserver for a signed attestation about the current time, including in it any nonces sent by Secondaries, so that Secondaries may trust that the time returned is at least as recent as their sent nonce
- generates a Vehicle Version Manifest with some vehicle metadata and all ECU Version Manifests received from Secondaries, describing currently installed images, most recent times available to each ECU, and reports of any attacks observed by Secondaries (can also be called directly: `dp.generate_signed_vehicle_manifest()`)
- sends that Vehicle Version Manifest to the Director (can also be called directly: `dp.submit_vehicle_manifest_to_director()`)

If you wish to run the demo with multiple vehicles (one Primary each), you can open a new
window for each vehicle's Primary and provide a unique VIN and ECU for each of them. Find the port that is chosen in the Primary's initialization and make note of it so that it can be provided to any Secondaries you set up in a moment (e.g. "Primary will now listen on port 30702")
For example:
```python
>>> import demo.demo_primary as dp

# Make sure the port matches the Primary's reported port, if there are multiple vehicles running.
>>> dp.clean_slate(vin='112', ecu_serial='PRIMARY_ECU_2', primary_port='30702') 
>>> dp.update_cycle()
```



### *WINDOW 5(+): the Secondary client(s):*
(The following assumes that the Image Repository, Director, Timeserver, and Primary have finished starting up and are hosting/listening.)
Here, we start a single Secondary ECU and generate a signed ECU Manifest
with information about the "firmware" that it is running, which we send to the
Primary.
```python
>>> import demo.demo_secondary as ds
>>> ds.clean_slate()
>>> ds.update_cycle()
```

Optionally, multiple windows with different Secondary clients can be run simultaneously. In each additional window, you can run the same calls as above to set up a new ECU in the same, default vehicle by modifying the clean_slate() call to include a distinct ECU Serial. e.g. `ds.clean_slate(ecu_serial='33333')`

If the Secondary is in a different vehicle from the default vehicle, this call should look like:
`ds.clean_slate(vin='112', ecu_serial='33333', primary_port='30702')`, providing a VIN for the new vehicle, a unique ECU Serial, and indicating the port listed by this Secondary's Primary when that Primary initialized (e.g. "Primary will now listen on port 30702").

The Secondary's update_cycle() call:
- fetches and validates the signed metadata for the vehicle from the Primary
- fetches any image that the Primary assigns us, validating that against the instructions of the Director in the Director's metadata, and against file info available in the Image Repository's metadata. If the image from the Primary does not match validated metadata, it is discarded.
- fetches the latest Timeserver attestation from the Primary, checking for the nonce this Secondary last sent. If that nonce is included in the signed attestation from the Timeserver and the signature checks out, this time is saved as valid and reasonably recent.
- generates an ECU Version Manifest that indicates the secure hash of the image currently installed on this Secondary, the latest validated times, and a string describing attacks detected (can also be called directly: `ds.generate_signed_ecu_manifest()`)
- submits the ECU Version Manifest to the Primary (can also be called directly: `ds.submit_ecu_manifest_to_primary()`)




### *Delivering an Update*
To deliver an Update via Uptane, you'll need to add the image file to the Image Repository, then assign it to a vehicle
and ECU in the Director Repository. Then, the Primary will obtain the new files, and the Secondary will update from the
Primary.

Perform the following in the **Image repository's** window to create a new file, add it to the repository, and host
newly-written metadata:

```python
>>> new_target_fname = filepath_in_repo = 'file5.txt'
>>> open(new_target_fname, 'w').write('Fresh target file')
>>> do.add_target_to_oemrepo(new_target_fname, filepath_in_repo)
>>> do.write_to_live()
```

Perform the following in the **Director repository's** window to assign that Image file to vehicle 111, ECU 22222:
```python
>>> new_target_fname = filepath_in_repo = 'file5.txt'
>>> ecu_serial = '22222'
>>> vin = '111'
>>> dd.add_target_to_director(new_target_fname, filepath_in_repo, vin, ecu_serial)
>>> dd.write_to_live(vin_to_update='111')
```

Next, you can update the Primary in the Primary's window:
```python
>>> dp.update_cycle()
```

When the Primary has finished, you can update the Secondary in the Secondary's window:
```python
>>> ds.update_cycle()
```

You should see an Updated banner on the Secondary, indicating a successful, validated update.



### *Blocking Attacks*
Uptane is designed to secure the software updates delivered between repositories and vehicles.  Section
7.3 of the [Uptane Design Overview](https://docs.google.com/document/d/1pBK--40BCg_ofww4GES0weYFB6tZRedAjUy6PJ4Rgzk/edit?usp=sharing) covers all of the known attacks in more detail.  We begin this section with a demonstration
of the Arbitrary Package Attack.


#### *Running an Arbitrary Package Attack on the Director repository without Compromised Keys*
This is a simple attack simulating a Man in the Middle that provides a malicious image file. In this attack, the
attacker does not have the keys to correctly sign new metadata (and so it is an exceptionally basic attack).

In the **Director's** window, run:
```python
>>> dd.attack_mitm(vin, new_target_fname)
```

Now, in the **Primary's** window, run:
```python
>>> dp.update_cycle()
```

Now, when the Primary runs dp.update_cycle(), it'll display the Defended banner and play a sound clip, as it's
able to discard the manipulated file without even sending it to the Secondary.

If you want to resume toying with the repositories, you can run a script to put the repository back in a
normal state (undoing what the attack did) by running the following in the Director's window:
```python
>>> dd.recover_mitm(vin, new_target_fname)
```

If the primary client runs an update_cycle() after the restoration of the Director repository, file5.txt
should update successfully.

To manually demonstrate the arbitrary package attack, issue the following commands in the Director console:
```python
>>> new_target_fname = 'file5.txt' # filename of file to create
>>> open(new_target_fname, 'w').write('Director-created target') # you could use an existing file, of course
>>> filepath_in_repo = 'file5.txt' # The path that will identify the file in the repository.
>>> ecu_serial = '11111' # The ECU Serial Number of the ECU to which this image should be delivered.
>>> vin = '111' # The VIN of the vehicle containing that ECU.
>>> dd.add_target_to_director(new_target_fname, filepath_in_repo, vin, ecu_serial)
>>> dd.write_to_live()
```
As a result of the attack above, the Director will instruct ECU 11111 in vehicle 111 to install file5.txt
Since this file is not on (and validated by) the Image Repository, the Primary will refuse to download it
(and a Full Verification Secondary would likewise refuse it even if a compromised Primary delivered it
to the Secondary).


#### *Running an Arbitrary Package Attack on the Image repository without Compromised Keys*

```
>>> do.arbitrary_package_attack(new_target_fname)

>>> dp.update_cycle()
```


The primary client is expected to discard the malicious `file5.txt` downloaded from the Image repository,
and only download the valid version of the file from the Director repository.

```Python
Update failed from http://localhost:30301/targets/file5.txt.
BadHashError
Failed to update /file5.txt from all mirrors: {u'http://localhost:30301/targets/file5.txt': BadHashError()}
Downloading: u'http://localhost:30401/111/targets/file5.txt'
Downloaded 17 bytes out of the expected 17 bytes.
```

Undo the the arbitrary package attack so that subsequent sections can be reproduced as expected.

```
>>> do.undo_arbitrary_package_attack(new_target_fname)
```

#### *Running a Rollback Attack without a compromised Director key*

We next demonstrate a rollback attack, where the client is given an older (and previously trusted)
version of metadata.

First, switch to the Director window and copy `timestamp.der` to `backup_timestamp.der`
A function is available to perform this action:
```
>>> dd.backup_timestamp(vin='111')
```                                                                     

A new `timestamp.der` and `snapshot.der` can be written to the live Director repository.                                 
```
>>> dd.write_to_live()                                                           
```

The primary client now performs an update...                                                                                
```
>>> dp.update_cycle()                                                            
```                                                                             

Next, move `backup_timestamp` to `timestamp.der` (`timestamp.der` is saved to               
`current_timestamp.der` so that it can later be restored), effectively rolling
back the timestamp file to a previous version.
```
>>> dd.rollback_timestamp(vin='111')
```

The primary client may now perform an update cycle, which should detect the rollback attack.         
```
>>> dp.update_cycle()
...
Update failed from http://localhost:30401/111/metadata/timestamp.der.
ReplayedMetadataError
Failed to update timestamp.der from all mirrors:
{u'http://localhost:30401/111/metadata/timestamp.der': ReplayedMetadataError()}
```

Finally, restore `timestamp.der`.  The valid, latest version of timestamp is moved back into place.  
```
>>> dd.restore_timestamp(vin='111')
``` 
 


#### *Running an Arbitrary Package Attack with a Compromised Director Key*

To start, add a new file to the image and director repositories.
```
>>> do.add_target_and_write_to_live(filename='evil', file_content='original content')
```

The new file is also added to the director repository.  We clear the previously
added target file so that the secondary client correctly installs a single target/firmware file.
```
>>> dd.clear_vehicle_targets(vin='111')
>>> dd.add_target_and_write_to_live(filename='evil', file_content='original content',
    vin='111', ecu_serial='22222')
```

To simulate a compromised directory key, we simply sign for a new "evil" version of the original file.
```
>>> dd.add_target_and_write_to_live(filename='evil', file_content='evil content',
    vin='111', ecu_serial='22222')
```

The primary client now attempts to download the malicious file.
```
>>> dp.update_cycle()
```

The primary client should print a "Defended" banner and provide the following error message: The Director has instructed
us to download a file that does  does not exactly match the Image Repository metadata. File: '/evil'



#### *Compromise the Image repository to also serve the arbitrary package*
```
>>> do.add_target_and_write_to_live(filename='evil', file_content='evil content')
```

Finally, the primary and secondary are updated.  Note, both the image and director repositories have been
compromised.  The primary installs the "evil" file, however, the secondary does not.

```
>>> dp.update_cycle()
>>> ds.update_cycle()
```

The secondary should detect that a malicious file was installed.  Since
both director and image repositories were compromised, the client would normally
be unable to detect this attack.  For demonstration purposes, the secondary ECU
in the demo code prints a banner indicating that the "evil" file was malicously 
installed: A malicious update has been installed! Arbitrary package attack successful:
this Secondary has been compromised! Image: 'evil'



#### *Recover from the compromised Director or Image keys*

We first restore the compromised repositories by reverting them to a
previously known, good state.  For the demo, this can be
accomplished by restarting the affected repositories and beginning with
a clean slate.  Thereupon, the compromised keys can then be revoked.

In the image repository window:
```
>>> do.kill_server()
>>> exit()
$ python
>>> import demo.demo_oem_repo as do
>>> do.clean_slate()
>>> do.revoke_and_add_new_key_and_write_to_live()
```

And in the director repository window:
```
>>> dd.kill_server()
>>> exit()
$ python
>>> import demo.demo_director as dd
>>> dd.clean_slate()
>>> dd.revoke_and_add_new_key_and_write_to_live()
```

#### *Restore the Primary and Seconday ECUs*

```
>>> dp.clean_slate()
>>> ds.clean_slate()
```

#### *Running another arbitrary package attack*
```
>>> do.add_target_and_write_to_live(filename='file6.txt', file_content='new content')
>>> dd.add_target_and_write_to_live(filename='file6.txt', file_content='new content', vin='111', ecu_serial='22222')
>>> do.arbitrary_package_attack('file6.txt')
>>> dp.update_cycle()
>>> do.undo_arbitrary_package_attack('file6.txt')
```

The primary client should again discard the malicious "file6.txt" file provided by the image repository.
