# *Joplin Text Notes and To-Dos via Speech*
##### *Voice memos recorded from the microphone, transcribed offline to text and converted to Joplin notes or To-Do tasks with automatic notifications. Can also transcribe batches of existing voice memos.*
---
_(This repository expands on the older [NoteWhispers](https://github.com/QuantiusBenignus/NoteWhispers) by bringing new tools, (bugs) and functionality, such as recording Joplin to-do tasks with automatic alarms and the ability to handle multiple voice memos. 
The note transcription utility **vm** is mature and will not see many changes.
The to-do utility **td** has the added complexity of parsing fuzzy datetime references with ongoing development of the code. 
Very much work in progress, the zsh version of `td` is ahead of the bash version in terms of pizzaz.)_

![vmJoplin-todo.png](resources/vmJoplin-todo.png)
![whisper-todo.png](resources/whisper-todo.png)

#### DESCRIPTION:  
These two Linux command-line utilities (with optional GNOME integration) are named respectively **vm** and **td** for brevity and quick access from the command line (check your PATH for conflicts and rename accordingly if needed.)

**vm** and **td** utilize previously unavailable, high-quality **offline** automatic speech recognition (ASR) technology (a derivative of [Open AI's](https://openai.com/) open-sourced [Whisper ASR models](https://github.com/openai/whisper)) to convert user speech, such as voice memos captured from the microphone (or pre-recorded audio files), into textual notes that are  automatically saved in the awesome, **open-source, note-taking application** [Joplin](https://joplinapp.org/). At its core, each utility records a voice memo from the default audio input channel (microphone) or uses one or multiple audio files as the input,  transcribes it into text using [whisper.cpp](https://github.com/ggerganov/whisper.cpp) (a C/C++ port of Open AI's Whisper) and either: 
   - **sends it (properly formated) to the clipboard**, or
   -  **creates a new note in a running instance of the Joplin note-taking app**  using Joplin's data API, or 
   -   **creates a new to-do task in the Joplin running instance** (in the case of **td**),  parsing the transcribed text for a valid datetime to set a notification/alarm (see details below)
   -   if Joplin is not running, **stores the note or to-do task in a file for later collection**.
   -   On a next invocation of either utility, if Joplin is up, **the temporarily stored notes / to-dos are collected** 
  
![td-mermaid-diagram.png](resources/td-mermaid-diagram.png)

As CLI scripts relying on built-in Linux tools under the hood (plus a few optional but common utilities such as `sox` and `curl`), **vm**'s and **td**'s feature set is exposed by a few command line arguments:
#### SYNOPSIS:
`vm [-b|-c|-bc|-cb|-h|--help] ... [filenames]`

`td [-b|-c|-bc|-cb|-h|--help|-a datetimespec] ... [filename(s)]`

- `vm` the default: use the 'tiny' whisper.cpp ASR model file and create a note in Joplin 
- `td` the default: create a to-do note in Joplin 
- `vm -b|--base` transcribes to a Joplin note using the larger (more accurate but slower) 'base' model
- `td -b|--base` transcribes a to-do task using the 'base' model
- `vm -c|--clip` will transcribe and output to the clipboard
- `td -c|--clip` will also output to the clipboard but text formated as inline to-do task
- `td -a|--alarm datetimespec` will create a to-do task with alarm set to trigger on 'datetimespec'
- `vm -bc` or  `td -bc , td -cb , td -[cb]a datetimespec` etc. - valid compound options are acceptable
- `vm -h|--help`  or `td -h|--help` will print help info
- any and all non-option arguments are treated as input audio files to be converted
      * *(tested on Ubuntu 22.04 LTS under Gnome version 42.5, with the English language models )*
---
Please, note that these 2 command line utilities for **Linux** are written for *zsh* but a quick (and possibly dirty) translation to bash (see folder For_bash_users) is provided for users of *bash*, who should use those instead of the zsh originals.


#### PREPARING THE ENVIRONMENT

##### PREREQUISITES:
* Joplin Linux desktop installation with the WebClipper plugin enabled (see https://joplinapp.org/) 
*  Whisper.cpp installation (see https://github.com/ggerganov/whisper.cpp)
*  Recent versions of 'sox', 'curl', 'jq', 'xsel' command line utilities from your system's repositories.
*  A working microphone (in GNOME one can set a keyboard shortcut to turn it ON and OFF )
> *DISCLAIMER: Setting up the environment for this to work requires a bit of attention and, quite likely for the novice user, reading about the Linux internals and making informed choices. Some of the proposed actions, if implemented, will alter how your system works internally (e.g. systemwide temporary file storage and memory management). The author neither takes credit nor assumes any responsibility for any outcome that may or may not result from interacting with the contents of this document.*
##### CONFIGURATION
Inside each script, near the begining, there is a clearly marked section, named **"USER CONFIGURATION BLOCK"**, where all the user-configurable variables (described in the following section) have been collected. Some can be left as is but others will require to be set to the user-specific values as determined by the specific instance of Joplin.
##### Temporary directory and files
*(NB. Everything in this section is based on the author's choice and opinion and may not fit the taste or the particular situation of everyone; please, adjust the script as you like. )*

Audio-to-text transcription is memory- and CPU-intensive task and fast storage for read and write access can only help. That is why **vm** and **td** are designed to store temporary and resource files in memory, for speed and to reduce SSD/HDD "grinding": `TEMPD='/dev/shm'`. 
This mount point of type "tmpfs" is created in RAM (let's assume that you have enough, say, at least 8GB) and is made available by the kernel for user-space applications. When the computer is shut down it is automatically wiped out, which is fine since we do not need the intermediate files.
In fact, for Joplin and any other applications (Electron-based or not) that are stored in Appimage format, it would be beneficial (IMHO) to have the systemwide /tmp mount point also kept in RAM. Every time you start Joplin, it expands itself in /tmp writing about 500 MB to your SSD or HDD and moving /tmp to RAM may speed up application startup a bit. A welcome speedup for any Electron app.  In its simplest form, this transition is easy, just run:
```
echo "tmpfs /tmp tmpfs rw,nosuid,nodev" | sudo tee -a /etc/fstab
```
and then restart your Linux computer.
For the aforementioned reasons, the scripts also expect to find the ASR model files needed by whisper.cpp in the same location (/dev/shm). These are large files, that can be transferred to this location at the start of a terminal session (or at system startup). This can be done using your .zshrc (or .bashrc) file by placing something like this in it: 
```
([ -f /dev/shm/ggml-tiny.en.bin ] || cp /path/to/your/local/whisper.cpp/models/ggml* /dev/shm/)
```

#### "INSTALLATION"
*(Assuming whisper.cpp is available and the "main" executable compiled with 'make' in the cloned whisper.cpp repo. See Prerequisites section)*
* Place the scripts **vm** and **td**  (or, for bash users, the scripts found in the "For_bash_users") somewhere in your PATH. 
* Create a symbolic link (the code expects 'transcribe' in your PATH) to the compiled "main" executable in the whisper.cpp directory. For example, create it in your $HOME/bin> with 
```ln -s /full/path/to/whisper.cpp/main $HOME/bin/transcribe```.
* Edit your personal NOTEBOOK_ID and AUTH_TOKEN variables in the code using the values from your Joplin app (see next section).  

If you are using the GNOME integration (recommended), don't forget to:
* Place `SpokenNotes.desktop` in `$HOME/.local/share/applications/
* Replace USERNAME and YOURPROFILENAME in the file with your values.
* Move the icon referenced in the .desktop file to the specified directory in $HOME/.local/...
* Find "Whispers" in your Activities and click "Add to Favorites" to pin it to the dock
* Create a new profile in gnome-terminal and edit it to suit your taste. Use its name in the .desktop file

##### Other environment variables

If Joplin is not running while the voice memo is being captured and transcribed, the scripts store transcribed notes or to-dos in the Joplin configuration directory for later processing (you can change this location as needed). This is in the code: `JOPLIND=$HOME'/.config/joplin-desktop/resources' `

The next two variables are for the Joplin data API. (***Please, replace with your own values from the Joplin desktop app for Linux***)

The first parameter is the id of the Joplin notebook where the new note will be created.
`NOTEBOOK_ID="PLACE_HERE_NOTEBOOK_ID_FROM_JOPLIN_RIGHT_CLICK_ON_NOTEBOOK_NAME"`
Of course, one can have these point to two different notebooks in **vm** and **td** to file ordinary notes separatelly from to-do tasks. 

The second variable is the authentication token generated by the Web Clipper plugin in Joplin. (Make sure web clipper is enabled, the token is needed to successfully interact with the REST API).
`AUTH_TOKEN="PLACE_HERE_TOKEN_FROM_JOPLIN_TOOLS_OPTIONS_WEB_CLIPPER_ADVANCED_OPTIONS"`

---
#### DETAILS
Sox is recording in wav format at 16k rate, the only currently accepted by whisper.cpp:
`rec -t wav $ramf rate 16k silence 1 0.1 3% 1 3.0 4% `
It will attempt to stop on silence of 2s with threshold of 6%, but you can always press CTRL-C to stop it manually. This is the only intervention that may be needed. 
After the memo is captured, it will be passed to `transcribe` (whisper.cpp) for speech recognition.
This will take a couple of seconds (fewer on a computer with fast CPU). One can adjust the number of processing threads used by adding  `-t n` to the command line parameters of transcribe (please, see whisper.cpp documentation). After transcription, the text is stored in a .txt file (-otxt argument in `transcribe -m $model -f $ramf -otxt`), in this case /dev/shm/vmfile.txt . 

The script will then format the data in the appropriate format (JSON for note creation via the data API) and send it to the desired output. If note creation was requested, a check will be made whether the REST API is exposed by the Web Clipper server (i.e. Joplin is running). If not, the JSON data will be stored in a `{timestamp}.json`  file to be picked up  on a later invocation of the script, when Joplin is running.

If a to-do task is being created, there is an additional intermediate step (quite a few steps actually) to be taken before contacting the API:

#### PARSING THE TRANSCRIBED TEXT FOR A TIME REFERENCE (in **td** - to set up a to-do alarm)

>*(N.B. Only spoken English time constructs, operation in the user's current locale and time zone.)*

If explicit datetime is not supplied, the transcribed text is parsed for a valid notification/alarm datetime.
It is quite difficult for computers to parse our spoken time references and using only built-in tools (i.e. coreutils date -d) presents a huge challenge when parsing arbitrary datetime text.
There are dedicated, complex NLP tools that work better but they are not perfect either.

That is why, to make things a bit easier, a keyword is used to separate the note body from the date-time reference to be parsed. This keyword can be used in the note body freely, it is the last instance within the text that is considered as the separator. For example, if the keyword is **"notification"** (this is user-configurable), then the last "notification" in the transcribed text is used to isolate the time reference:
For example:
* *"Need to see my dentist next week. Set **notification** for Tuesday"*       - this is valid.
* *"Scheduled a company meeting with **notification** for 2023/5/24 at 8pm"*   - also OK.
* *"Guests need prior notification. Set one **notification** for March the 3rd in the evening."*  -OK
* *"...**notification** for next week"*
* *"...**notification** in 3 hours"*
* *"...**notification** tomorrow morning"*  (see source code for "morning" & other adjustable definitions)
* *"...**notification** in 33 hours and 5 minutes"*
* *"...**notification** on the ninth month +1000 seconds"*
... are all valid.
* or even *"...**notification** at the usual time"* for some extra customization (see code for ideas).

Speaking literaly "YYYY/MM/DD", followed by time (if needed) e.g. *"2024 slash 5 slash 23 at 1pm"* works well too.
As a minimum, the month should precede the date and time, e.g. "March 12" not "12 of March".
If parsing is unsuccessful, the utility will not set an alarm in Joplin and it has to be done manualy. 
A warning will be issued but the to-do task will be created successfully. 
The failure can be due to errors in the user instructions, errors in the speech recognition, limitations of the simplistic datetime preprocessor etc. With practice (and good diction:-) the error rate can be comparable to the error rate for speech recognition.
In some edge cases, successful parsing gives incorrect datetime. Some practice needed to avoid those
For scheduling critically-important stuff with this utility, use the command-line option "-a"
and provide explicit *datetime specification* or instead, simply set the to-do alarm time in Joplin.

#### Gnome desktop integration
To make interaction with this CLI utility more convenient, one can create a GNOME desktop entry (if using GNOME) with a custom profile for the terminal window (small window, custom color, transparent, etc., see `gnome-terminal` documentation on creating named profiles ) so that the window will be visible on top a maximized Joplin window. 
One can also choose whether to keep the terminal window open, or close it after the transcription (see the gnome-terminal settings for your custom profile - YOURPROFILENAME in the code below.)
Sample `SpokenNotes.desktop` (Replace USERNAME and YOURPROFILENAME and place in your ` $HOME/.local/share/applications/`):
```
[Desktop Entry]
Name=Spoken_Notes_To-Dos
Comment=For use with the vm and td CLI utilities
Exec=gnome-terminal --window-with-profile=YOURPROFILENAME --hide-menubar --geometry=64x6+380+920 --title=Speech-to-Joplin
Icon=/home/USERNAME/.local/share/icons/hicolor/128x128/apps/mic128.png
Terminal=true
Type=Application
Categories=Application
Actions=new-note;new-clip;new-todo;inline-todo;

[Desktop Action new-note]
Name=New Joplin Note
Exec=gnome-terminal --window-with-profile=Lilico --hide-menubar --geometry=64x6+380+920 --title=NewNote -- vm

[Desktop Action new-clip]
Name=Record To Clipboard
Exec=gnome-terminal --window-with-profile=Lilico --hide-menubar --geometry=64x6+380+920 --title=NewClip -- vm -c

[Desktop Action new-todo]
Name=New Joplin ToDo
Exec=gnome-terminal --window-with-profile=Lilico --hide-menubar --geometry=64x6+380+920 --title=NewTodo -- td

[Desktop Action inline-todo]
Name=ToDo to Clipboard
Exec=gnome-terminal --window-with-profile=Lilico --hide-menubar --geometry=64x6+380+920 --title=InlineTodo -- td -c

```
With the above `gnome-terminal` desktop entry ( please, adjust profile and username), the  utility will be accessible from the system dock, after you add it to your "Favorites" (right mouse click brings up the shown context menu):

>![whispers-menu3.png](resources/whispers-menu3.png)


The .desktop entry is set so that just clicking on the dock icon with the left mouse button will open the terminal and wait for a command (such as `td -ba "2023 March 23 23:00"`), while invoking one of the context menu commands will immediatlely start recording and will close the window when finished transcribing the note / to-do. The "tiny" ASR model file is used by default in the desktop menu actions but that can be changed as desired in the .desktop file by adding  the `-b` option to the respective command.

If using X11 (instead of the restrictive Wayland), one can use the `--geometry` command line argument to position the small terminal window  in front of a dead space in the Joplin window and set it to stay on top (screenshots):



![vmJoplin.png](resources/vmJoplin.png)


Even with the default "tiny" model, the accuracy  (English language tested) is impressive and on a faster computer (not mine) it takes less than a second to transcribe a 30s audio clip with essentially no errors. As such, this command-line utility, combined with the power of [Whisper](https://github.com/openai/whisper) from Open AI (its [whisper.cpp](https://github.com/ggerganov/whisper.cpp) port, to be more precise), proves quite useful and practical, especially in the context of a note-taking app such as the versatile, customizable [Joplin](https://joplinapp.org/). Enjoy!

### Credits
* Open AI (for [Whisper](https://github.com/openai/whisper))
* Georgi Gerganov and community ( for Whisper's C/C++ port [whisper.cpp](https://github.com/ggerganov/whisper.cpp))
* Laurent Cozic and community (for the [Joplin](https://github.com/laurent22/joplin) note-taking app)
* The **curl** developer community (for the versatile and powerful **[curl](https://github.com/curl/curl)**)
* The **sox** developers (for the venerable "Swiss Army knife of sound processing tools")
* Stephen Dolan and community (for **jq**, *"the sed for JSON"*)
* The creators and maintainers of old and new utilities such as **xsel, xclip**, the heaviweight **ffmpeg** and others that make the Linux environment (CLI and GUI) such a powerful paradigm.

