<div align=center><img src="https://github.com/user-attachments/assets/50bb9d9d-7c02-49b0-acc3-a3cdbcb1d40e" height="71" width="345"></div><br>

<hr>

A Note to Seasoned Developers
-
Roblox's moderation has failed developers in so many ways. Assets on the creator store are consistently taken down for illogical and unfounded reasons. It's now impossible to publicly distribute a module that can be dynamically updated through a simple require(ID), unless you're one of the few lucky creators whose asset was selected as "pre-approved" before Roblox dropped the update back in 2022 that destroyed all systems that rely on being able to update code remotely, forcing developers to manually update the software each time that an update was published. As a result of this, ReSync is loaded into your game using unconventional means. Roblox actively stops any and all attempts to use require(ID) or InsertService:LoadAsset(ID) in public modules by pulling them off the creator marketplace entirely. While this is admittedly very inconvenient, there's one final, albeit performance-intensive workaround that can be applied to bypass this structure entirely, not relying on any built-in Roblox methods other than those detailed in HttpService.

How it Works
-
The purpose of the ``Loader`` script inside of the model is to act as a boot program for ReSync. Ideally, this boot process would be unnecessary, instead pulling the source directly from a ``MainModule`` via the use of ``require(Id)``, but as explained earlier, this is sadly no longer possible. At the top of the loader is the path to the ``Settings`` ModuleScript. This can be changed by a developer who wishes to place the settings module in a separate location from the default file structure. This would be strange, but some people are picky about where their stuff is stored for organizational purposes. First, the loader begins running a benchmark. This is done to monitor how long it takes for the system to initialize, and is done by calling the built-in ``os.clock()`` function, which returns the number of seconds passed since the CPU first started. Since this is running on the server, it returns the amount of time passed since the Server's CPU began running. This number isn't always zero when the server first boots, indicating that a "game server" may not be a phycial Roblox server. Regardless, this number being arbitrary does not negatively impact the purpose the loader is utilizing it for, so long as it consistently continues to count from that epoch with sub-second precision. As explained previously, two values, those being the locations of the ReSync directory ``project`` and the settings ModuleScript ``settingsModule``, are declared so that the end user has control over where they place different parts of the system. Following this, the loader script itself is sent into variable ``loader`` for ease of access. The next thing that the loader does is to initialize the table ``bootLog``, which is an array of strings that will be sent to the internal program as part of a series of "external" information. More on this later. After this, the loader retrieves a variety of dependencies, which are detailed below:

| Dependency                      | What it Does |
| ------------------------------- | ------------ |
| HttpService ``http``            | Roblox service to handle HTTP requests
| DataStorage ``dModule``         | Support module for interfacing with data stores in Roblox
| InternalFlags ``iFlags``        | Static feature flags reserved as part of the internal SDK to quickly toggle portions of code
| DefaultSettings ``defaultSets`` | ReSync script setting defaults in the absence of a setting, or mismatched type; not to be confused with the settings that can be edited from in-game
| Interpreter ``wrap``            | Lua bytecode interpreter, modified for ReSync purposes
| InputStream ``inStm``           | Handles the reading of binary data
| LibDeflate ``balloon``          | String compression and decompression
| Serial ``serial``               | Manipulation of binary data into data storage, and reversing that process for execution

Following this step, the error function is modified to display ReAync's tag and set the level to 0. From here, the system checks to see if the location defined at the top of the script as the variable ``settingsModule`` is valid, and if it is a ModuleScript containing a table. If not, the system marks that default settings will be loaded. Then, the local data storage dependency is loaded into the variable ``dModule``. Next, ``settings.DataCategory`` is validated, and if it does not exist or is not a string, the system defaults to "``ReSync``". The data stored within said category is retrieved with ``dModule:GetCategory(category)``. If the file system is unable to be found or is corrupted (more on the latter shortly), an additional step will be performed here, which is detailed below. Otherwise, see **<a href="./Loader.md#init">init</a>**.

## Archive Download
In the event that ReSync must be obtained from the endpoint, the following steps are performed:
1. The Roblox engine checks HttpEnabled (because we finally have read access to that property instead of having to send a GET to google.com), and if not, throws an error.
2. An HTTPS request is sent to this repository at ``Distribution/LatestVersion.txt`` to determine which build should be downloaded.
3. A fix is applied to the returned string, removing the termination character (0x10). As this character is invisible, it will not appear in the string if printed, which actually caused significant confusion and delay in development due to it being so difficult to spot.
4. An HTTPS GET is sent to ``Distribution/{iFlags.SourceName}_{latest}.rsarc?raw=true``. The RSArc file extension stands for ReSync Archive, and is a proprietary binary format.
5. Jump to **<a href="./Loader.md#init">init</a>**.

## init
``init`` is the main function in the loader, and it serves as the entry point into the system internals.

1a. The RSArc is parsed into a stream object for binary reading.

1b.The loader will first read an eight byte double from the input stream. This number is the string length of the compressed bootstrap program.

1c. The previously determined number is extracted from the stream and decompressed.

2a. Alternatively, if ``settings.LocalBuild`` is set to true, the system will build itself here. As the compiler is not natively included with the release version of ReSync, this will throw an error if attempted by the end user.

3. The environmental variables are stored for internal use.

4. The compiled bootstrap file is wrapped into an executable function, ``ReSync``. There is one caveat to this - Roblox deletes assets from the creator marketplace that utilize the only two functions capable of environment retrieval and manipulation, those being ``getfenv`` and ``setfenv``. It is for this reason that there is a property in the script settings called "Environment", which is commented out by default, and it's up to the end user to remove that comment in order to permit the system to load.

5. ReSync is called with no arguments.

# TODO
32-bit cyclic redundancy check, ensures that data is not corrupt or tampered with
