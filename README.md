Speech Synthesis Interface Protocol
===================================

The [Speech Synthesis Interface Protocol][SSIP] (SSIP)
defines a set of text-based commands and responses
between a server and a client
for requesting speech to be synthesised and  played to end users.
Its specification is available in a [webpage format][ssip webpage].

There is a de-facto reference implementation of SSIP
called [speech dispatcher][], or speechd for short,
using the POSIX API in C.

Architecture
------------

An SSIP server typically provides an endpoint for clients to connect to.
Once connected, SSIP allows connected clients to send commands
to request messages to be spoken, control how new messages are spoken,
or query about past messages.

To speak a message to an end user, an SSIP server uses output modules.
Output modules are responsible for synthesising text to speech
and sending the speech data to an audio device.
(The SSIP does not actually mention how audio is output to the end user.)
They typically do this
by wrapping around external speech synthesiser libraries,
which in turn forward requests to the synthesisers
and configure them to output audio to end users.

Concurrent clients can use different output modules at the same time.
Output modules available for clients to choose from and how they are used
are determined by the SSIP server.
A plugin architecture is used by speechd to make it extensible,
but implementations are free to decide how output modules are found.
In particular, it may provide only a single built-in output module.

In this sense, an SSIP server essentially acts as an intermediary
rather than directly providing a speech service.
Its minimal responsibility is to schedule messages from clients,
and dispatch them to output modules using the correct settings.

Objects
-------

To perform its work, an SSIP server has to
keep track of clients, messages, output modules,
and manage all their states.

### Clients

A client in SSIP is a connected session.
If a client reconnects, it is treated as a new client.

Every client is associated with  a set of configurations
on how to speak a message.
For example, what the voice should sound like
or how it should be played to the end user.
The configuration is used for all new messages received from the client
until the configuration is changed.

Different clients can have different configurations
which are discarded on disconnect.
So clients need to set any non-default configurations again
after they reconnect.

### Messages

A message is a speech request from a client.
The request can be to speak a piece of text,
or to play a text icon which is a short sound clip,
typically used to indicate, for example, a capital letter or end of file.

Accompanying the message are descriptions
of how the message should be played.
These are inherited from the client
when the message request is first received.

And amongst the descriptions or states is a priority.
Since only at most one message can be played at any point in time,
message priorities are used to determine what happens
when there are more than one messages waiting to be played.
They may start playing immediately, postponed,
or cancelled from playing completely.

An SSIP server keeps a history of messages received
unless requested not to do so.
This history allows the end user to query past messages
and to replay them if missed, or for clarifications.
This means all messages, whether played or not,
are usually be stored on the server.

Building on top of messages,
the SSIP defines blocks as ordered sets of messages
that are treated as a single message
for history and scheduling purposes.
They allow messages to be treated as a single one
when they are postponed or stopped.
This matters when messages require each other for context,
so that the messages would be postponed and stopped together.
An example would be text icons in the middle of texts.

### Output Modules

An output module plays requested messages to end users.
The server is responsible for sending messages
to output modules in expected order and with settings,
and for cancelling speaking messages
if supported by the output module as appropriate.

One of the settings is its synthesis voice
which is a configuration for synthesising text to speech.
Each output module provides, to the SSIP server,
a set of synthesis voices that its synthesiser supports.
Each synthesise voice is associated with a language
so that end users can pick a voice by language as well.

Building on top of synthesis voices,
the SSIP defines a default set of voice types.
The voice types are meaningful to the end users
as a textual description of a voice.
This default set allows users to always have default fallbacks.
Output modules must be able to choose a synthesis voice
based on a given voice type and language.

An optional responsibility of output modules is event notification.
An event is a change of state in a message,
such as when it starts or stop to play,
or has reached specified points in the message.
Output modules may inform the server of such events.
The server can then forward such events to clients interested in them.

Server States
-------------

Each object tracked by an SSIP server
has a set of minimal state, data or settings,
include those of the server itself.
While the server need not store the states explicitly,
it must be able to deduce these information as necessary.
A server should keep track of the following:

-   `DEBUG` mode.
    If `ON`, all debug information are written into a file.
    If toggled from `ON` to `OFF`, then the file is closed.
-   `OUTPUT_MODULES` available to clients.
-   Known clients. (Connection is probably a better name.)
-   `HISTORY` of received messages, including cancelled messages.
    (I am unsure how to implement this,
    in terms of what information to keep.
    There are some contradictory things in the SSIP
    and it does not mention how to deal with blocks.)
-   Current message. The message being spoken if any.

Client States
-------------

Each client has a client ID.
It is a positive integer.
Unique for the lifetime of the server.

`CLIENT_NAME` is displayed to end users.
It is a string of the form `<user>:<client>:<component>`
where the three values consist of

-   The hyphen character `U+002D`,
-   The digit characters `U+0030`-`U+0039`,
-   The Latin capital letter characters `U+0041`-`U+005A`,
-   The underscore character `U+005F`, and
-   The Latin small letter characters `U+0061`-`U+007A`.

It may be any such strings,
but a convention is used to give them meaning.

-   The `user` value identifies the user of the client;
-   The `client` value identifies the client;
-   The `component` value identifies further partitioning
    of client functionality.

The name need not be unique.
It is provided for identification by end users,
filtering in server configuration
and message history manipulation.

This is set at most once by the clients themselves.
While setting it is technically optional,
clients should set a name
immediately after establishing connection,
or as soon as possible.

Each client has a `HISTORY` property.
It is a boolean value.
It determines whether to store, into message history,
new messages received from the client,
includes messages that are subsequently cancelled.

Client New Message States
-------------------------

These client states determine what settings to use
for new messages received from the client.
Changing them affects newly received messages.
In particular, messages received from the client
but are queued to be played or being played
are unaffected by such changes.

### Processing Options

These options determine how given message contents should be parsed
before being presented to the end users.

`CAP_LET_RECOGN` determines
how to indicate that a capital letter has been encountered.
It has three modes.

-   In `NONE` mode, capital letters are not given any special processing.
-   In `SPELL` mode, capital letters are spelt when encountered
    even when spelling mode is off.
    The letters spelt are determined by `CAP_LET_RECOGN_TABLE`.
    (This setting is not specified in the specification yet.)
-   In `ICON` mode, each capital letter is preceeded by a sound icon.

`PUNCTUATION` characters may be set to be spoken outloud in sentences.
This may be useful for proof-reading or programming.
It has four modes.

-   In `NONE` mode, punctuation characters are not spoken.
-   In `SOME` mode, output module determines which are spoken.
-   In `MOST` mode, output module determines which are spoken,
    but it should generally be more verbose than `SOME` mode.
-   In `ALL` mode, every punctuation character is spoken.

`SPELLING` of text messages may be toggled on in place of reading words.
This may be useful for proof-reading.

-   If `ON`, then messages will be read by letters.
-   If `OFF`, then messages will be read as words or sentences.

`SSML_MODE` determines whether text messages
uses SSML to provide extra speech contexts.
However, not all output modules parse such markup appropriately.

-   If `ON`, then message text are expected to be wrapped in SSML.
    For example, every message should be wrapped
    between `<speak>` and `</speak>` tags
-   If `OFF`, then plain texts are expected.

In speechd, some initial values can be set in `speechd.conf`.

-   The `DefaultCapLetRecognition` defaults to `none`.
-   The `DefaultPunctuationMode` defaults to `none`.
-   The `DefaultSpelling` setting defaults to `off`.

### Audio Adjustments

There are three options for making adjustments to audio output.
Their values are integers between -100 and 100 inclusive.
The values do not have well-defined meaning,
but going up and down have "expected" meanings for the settings.
(The specification does not specify whether they affect sound icons.)

`PITCH` provides pitch adjustment to synthesised speech.
The greater the value, the higher the pitch.

`RATE` determines how fast synthesised speech is.
The greater the value, the faster the speech.

`VOLUME` determines how loud audio output is.
The greater the value, the louder the audio output.

In speechd, some initial values can be set in `speechd.conf`.

-   The `DefaultPitch` setting defaults to `0`.
-   The `DefaultRate` setting defaults to `0`.
-   The `DefaultVolume` setting defaults to `100`.

### Voice Settings

There are four settings controlling the voice for speeches.

`OUTPUT_MODULE` determines which synthesiser to use.

`SYNTHESIS_VOICE` is the voice to speak new messages.

`VOICE_TYPE` is a voice description to try to satisfy.
The standard SSIP voice types are

-   `CHILD_FEMALE`, `CHILD_MALE`,
-   `FEMALE1`, `FEMALE2`, `FEMALE3`,
-   `MALE1`, `MALE2` and `MALE3`.

`LANGUAGE` is a voice language to try to set.

In speechd, some initial values can be set in `speechd.conf`.

-   The `DefaultVoiceType` setting defaults to `MALE1`.
-   The `DefaultLanguage` setting defaults to `en-US`.

When the synthesis voice is changed,
the given voice is used as is,
since it should be a voice known by the synthesiser.

When the voice type or language is changed,
the output module is responsible
for picking a new synthesis voice
based on the given voice type or language.
It should try to preserve the other setting if possible.
The synthesis voice is also changed to the chosen voice.
It need not do anything if the existing synthesis voice
already satisfy the newly given requirement.

When the output module is changed,
since synthesis voices are tied to output modules,
the current synthesis voice must be changed.
The new output module should pick a new synthesis voice
as if the current language has been set anew.

### State Transitions

`PRIORITY` of messages determines what happens
when there are multiple messages queueing to be played.
It can be one of the following five priorities.

-   `NOTIFICATION`
    -   Cancels itself
        if the current message or any queued messages
        are not of `NOTIFICATION` priority.
    -   Cancels all other messages
        of `NOTIFICATION` priority.
    -   So it is never queued.
-   `PROGRESS`
    -   Cancels itself
        if the current message or any queued messages
        are not of `NOTIFICATION` priority.
        (Use of `NOTIFICTION` is deliberate.)
    -   Cancels all other messages
        of `notification` priority.
        (Or all current messages,
        since `NOTIFICATION` messages are not queued.)
    -   (There is currently a behaviour described in the SSIP
        about playing the last progress message
        when none are playing.
        There seems to be some contradictions
        in its description.
        For now, only the above behaviours are assumed.)
-   `TEXT`
    -   Cancels all other messages
        of `TEXT`, `NOTIFICATION` and `PROGRESS` priorities.
    -   Queues if there is a playing message
        not of those priorities.
-   `MESSAGE`
    -   Cancels all messages
        of `NOTIFICATION`, `PROGRESS` or `TEXT` priorities.
    -   Queues if there is a playing message
        not of those priorities.
        It will be played before any queued `TEXT` messages.
-   `IMPORTANT`
    -   Cancels any current message
        of `MESSAGE` and `TEXT` priorities.
    -   Cancels all messages
        of `NOTIFICATION` and `PROGRESS` priorities.
    -   Queues if there is a playing `IMPORTANT` message.
        It will be played
        before any queued `TEXT` and `MESSAGE` messages.

`NOTIFICATION` events about speaking progress of messages
can be useful for determining what inputs to accept.
(This is unrelated to the `NOTOFICATION` priority.)
Without notification events,
most message requests are essentially fire-and-forget,
and there are no guarantees
as to when or if a message will be spoken.
There are six types of notifications possible.
They can be independently toggled on and off
for each message.

-   `BEGIN`: Message has started playing.
-   `END`: Message has finished playing to the end.
-   `CANCEL`: Message has stopped playing, and will not finish.
    Only one of `CANCEL` and `END` events
    can be emitted for each message.
-   `PAUSE`: Message is postponed.
    It can only come after a `BEGIN` event.
-   `RESUME`: Message is unpaused.
    It can only come after a `PAUSE` event.
-   `INDEX_MARK`: Message reading
    has reached a marked position.
    Only emitted if SSML mode is on for the message.

`PAUSE_CONTEXT` specifies the number of sentences to replay
when a message resumes from a pause.
If the already spoken parts are not long enough,
then speak the whole message from the beginning.
This can be helpful for reminding the listener
of where the message was paused at.
In speechd, the initial value can be set in `speeched.conf`
using the `DefaultPauseContext` setting
which defaults to `0`.

Message States
--------------

Each message has a message ID.
It is a positive integer.
Unique for the lifetime of the server independent of sender client.

Each message has a type.
It can be either a text, icon, key, character or a block.

Each message has content whose value is interpreted based on its type.

Each message has a received-at date and time with second resolution.

Each message may have a pause position
indicating where to continue speaking when its turn begins.
For example, it may be at the start if it had never begun.
It may be at the end if it had ended.
If may be in the middle of the text if it was paused.
It might be a good idea to make sure `PAUSE_CONTEXT`
does not cause the position to move backwards.

Each message may be queued.
This is independent of the pause position.
If a message is cancelled while paused,
then it may not be queued
while its position is in the middle of its text content.

Each message has output settings.
These settings are copied from the sender client.
They need to be copied since the client may change these settings
and the changes should not affect already received messages.

-   Client ID, recorded as the sender.
-   Processing options.
    -   `CAP_LET_RECOGN`
    -   `PUNCTUATION`
    -   `SPELLING`
    -   `SSML_MODE`
-   Audio adjustments.
    -   `PITCH`
    -   `RATE`
    -   `VOLUME`
-   Voice settings.
    -   `LANGUAGE`
    -   `OUTPUT_MODULE`
    -   `SYNTHESIS_VOICE`
    -   `VOICE_TYPE`
-   State transitions.
    -   `NOTIFICATION` list of events to emit
    -   `PAUSE_CONTEXT`
    -   `PRIORITY`

Output Module States
--------------------

Each output module has a name.
It is a string.
It is provided for identification by end users.
(The specification does not say whether the name needs to be unique.
The specification does not say which are valid name characters.)

`SYNTHESIS_VOICES` are provided by each output module.
It is a set containing synthesis voices
supported by the synthesiser managed by the output module.
The set needs not be exhaustive.
They are provided for identification by end users
for configuration purposes.
Each synthesis voice provided should be given

-   A name. It is a string. For identification by end users.
-   A language code. It is a string, compliant to RFC 1766.
-   Dialect. It is a string.

The Protocol
------------

Connected clients can send commands to the server
to request information or an action to be performed.
A corresponding response is then usually sent back
containing the requested information
or the status of the request.
A client is not allowed to send another command
until a complete response for the previous command has been received.

Commands sent by clients are of the form

    <command> [arg ...]\r\n

Commands and arguments are delimited by any number of spaces.
Commands and predefined constant-valued arguments are case-insensitive.
Currently available commands are as follows.

-   Message request: `CHAR`, `KEY`, `SOUND_ICON`, `SPEAK`.
-   Queue control: `CANCEL`, `PAUSE`, `RESUME`, `STOP`.
-   Setting manipulation: `LIST`, `GET`, `SET`.
-   Message grouping: `BLOCK`
-   Message query: `HISTORY`
-   Client commands: `HELP`, `QUIT`

Many commands has a `client` parameter
to refer to a target client to act on.
In such cases, the argument
may be the client ID,
the string `SELF` targetting the client sending the command,
or the string `ALL` targetting all connected clients.

Responses from the server are of the following form.

    [<return-code>-<result>\r\n ...]
    <return-code> <human readable message>\r\n

Return codes are three-digit numbers
and determine whether the last command was successful.
The value of result depends on the command sent.
The human readable message is for debugging purposes.

-   100-199 for a successful information request.
-   200-299 for a successful operation.
-   300-399 for an error from the server side.
-   400-499 for an error from an invalid client command.
-   500-599 for an error from an invalid client input (e.g. unparsable).
-   700-799 for an event notification.

The ten and unit digits are currently unused, except for 701-705.
Their values given in example responses cannot be depended on.

Message Request
---------------

These commands ask the server
to play a message to the end user if and when possible.

### Char Command

The `CHAR <char>` command requests the given character to be spoken.
The `char` value is a UTF-8 character to be spoken
or the string `space` if the request is for the space character `U+0020`.
The server responds with the new message ID if successful.

    CHAR X
    200-2
    200 OK CHAR QUEUED
    CHAR space
    200-3
    200 OK CHAR QUEUED

### Key Command

The `KEY <key>` command
requests the keys represented by the case-sensitive `key` value
to be spoken.
The server responds with the new message ID if successful.

The `key` value is a concatenation of strings,
each representing a key, delimited by underscores.
A key is presented by a UTF-8 character,
excluding certain control characters,
or a constant string typically for non-character keys.

Some keys are modifier keys.
Only one non-modifier key may be included in each `key` value
and it must be the last one in the sequence.
Available modifiers
are `alt`, `control`, `hyper`, `meta`, `shift` and `super`.

The excluded UTF-8 characters are

-   Control characters from `U+0000` to `U+001F`,
-   The space character `U+0020`,
-   The quotation mark character `U+0022`,
-   The underscore character `U+005F`, and
-   Invisible characters from `U+007F` to `U+009F`.

Some of the excluded code points are represented by constant strings:

| Code point | Key name       |
| :--------: | :------------- |
|  `U+0008`  | `backspace`    |
|  `U+0009`  | `tab`          |
|  `U+000A`  | `enter`        |
|  `U+000D`  | `return`       |
|  `U+001B`  | `escape`       |
|  `U+0020`  | `space`        |
|  `U+0022`  | `double-quote` |
|  `U+005F`  | `underscore`   |
|  `U+007F`  | `delete`       |

Other than the above, the remaining constants are as followings:

-   `break`, `pause`, `print`
-   `f1`, `f2`, ..., `f24`
-   `home`, `end`, `insert`, `next`, `prior`
-   `kp-*`, `kp-+`, `kp--`, `kp-.`, `kp-/`, `kp-enter`
-   `kp-0`, ..., `kp-9`
-   `menu`, `window`
-   `num-lock`, `scroll-lock`
-   `up`, `down`, `left`, `right`

Examples of `KEY` command.

    KEY 1
    200-4
    200 OK KEY QUEUED
    KEY alt_tab
    200-5
    200 OK KEY QUEUED
    KEY control
    200-6
    200 OK KEY QUEUED
    KEY control_shift_tab
    200-7
    200 OK KEY QUEUED

### Sound Icon Command

The `SOUND_ICON <icon-name>` command
requests a predefined sound to be played.
The `icon-name` parameter is a constant string,
either specified by the specification (currently none)
or any supported by the speech server configuration.
The server responds with the new message ID if successful.

    SOUND_ICON message
    200-8
    200 OK SOUND_ICON QUEUED
    SOUND_ICON not-found-icon
    400-NO SOUND_ICON UNKNOWN

### Speak Command

The `SPEAK` command requests text to be spoken.
It has no parameters.
If the server responds to this request with a success,
the client can start sending the text to be spoken.
(Mostly verbatim. See below.)
The text sent must be terminated by the string `\r\n.\r\n`.
After termination is detected,
the server responds with the new message ID if successful.

In the text to be spoken, lines are delimited by `\r\n`.
A line starting with a dot acts an escape.
In particular, any text lines that start with a dot
must be prepended with another dot to ignore the escape.
A line with a dot on its own denotes the end of the text,
hence the text must be terminated by `\r\n.\r\n`.
A successful response should be received after this termination marker.

    SPEAK
    200 OK SPEAK AWAITING TEXT
    Hello, world!
    .This is spoken.
    .
    200-1
    200 OK SPEAK QUEUED

Queue Control
-------------

These commands provide control over the message queue as a whole.
They are similar to some playlist controls.

### Cancel Command

The `CANCEL <client>` command
stops the current message if it was received from the given client.
All queued messages received from the given client
are popped and will not be played.
This effectively stops audio output from the given client
as soon as possible, and clears its queue.

    CANCEL SELF
    200 OK CANCEL DONE

### Pause Command

The `PAUSE <client>` command
stops the current message if it was received from the given client,
keeping track of its current position, to continue speaking from
when a `RESUME` command targetting a corresponding client is received.
All queued messages received from the given client remain queued,
but cannot be played until a `RESUME` command is received.

While paused, all messages received from the given client are queued.

    PAUSE SELF
    200 OK PAUSE DONE

### Resume Command

The `RESUME <client>` command 
allows messages received from the given client,
but withheld by a `PAUSE` command, to be played again.

This command fails if a corresponding `PAUSE` command was not sent.
(However, the SSIP does not indicate what happens
if something similar to `PAUSE SELF` and then `RESUME ALL` is done.
It may be prudent to consider it a success
as long as some client is resumed by the command.
And I am not sure if speechd actually considers it as an error.)

    PAUSE SELF
    200 OK PAUSE DONE
    RESUME SELF
    200 OK RESUME DONE
    RESUME SELF
    400 NO RESUME NOT PAUSED

### Stop Command

The `STOP <client>` command
stops the current message if it was received from the given client.
The message will not be played again.
Queued messages received from the client may be played as normal
depending on priority.

    STOP SELF
    200 OK STOP DONE

Processing Options
------------------

These settings determine how to parse content of new messages.

### Capital letters

The `SET <target> CAP_LET_RECOGN <mode>` command
changes if and how capital letters are identified.

    SET SELF CAP_LET_REGION SPELL
    200 OK SET CAP_LET_RECOGN DONE
    SET SELF CAP_LET_RECOGN NONE
    200 OK SET CAP_LET_RECOGN DONE

### Punctuation

The `SET <target> PUNCTUATION <mode>` command
changes which punctuations should be spoken.

    SET SELF PUNCTUATION ALL
    200 OK SET PUNCTUATION DONE
    SET SELF PUNCTUATION NONE
    200 OK SET PUNCTUATION DONE

### Spelling

The `SET <target> SPELLING <bool>` command
changes whether to read by letters or by words.

    SET SELF SPELLING ON
    200 OK SET SPELLING DONE
    SET SELF SPELLING OFF
    200 OK SET SPELLING DONE

### SSML markup

The `SET SELF SSML_MODE <mode>` command
changes whether the server should expect SSML
from the client sending the command.

    SET self SSML_MODE on
    SPEAK
    200 OK SPEAK AWAITING TEXT
    <speak>
    Hello, world!
    </speak>
    .
    200-9
    200 OK SPEAK QUEUED

Audio Adjustments
-----------------

The way audio is output to the end user can be changed
to make it more comfortable to listen to,
especially when it is used for long periods of time.

### Pitch

The `GET PITCH` command returns
the current pitch adjustment value as an integer.

    GET PITCH
    200-0
    200 OK GET PITCH DONE

The `SET <client> PITCH <value>` command
changes the voice pitch of message received from the given client.

    SET SELF PITCH 8
    200 OK SET PTICH DONE

### Rate

The `GET RATE` command returns the current rate as an integer.

    GET RATE
    200-0
    200 OK GET RATE DONE

The `SET <client> RATE <value>` command
changes the speech rate of messages received from the given client.

    SET SELF RATE 16
    200 OK SET RATE DONE

### Volume

The `GET VOLUME` command returns the curent volume as an integer.

    GET VOLUME
    200-100
    200 OK GET VOLUME DONE

The `SET <client> VOLUME <value>` command
changes the audio volume of messages received from the given client.

    SET SELF VOLUME 50
    200 OK SET VOLUME DONE

Voice Setting
-------------

The following commands change the voice used to produce speech.

### Language

The `SET <client> LANGUAGE <language>` command
changes the synthesis voice for new messages received from the given client
to one compatible with the given language
if it is not already compatible.

    SET SELF LANGUAGE en
    200 OK SET LANGUAGE DONE

### Output Module

The `LIST OUTPUT_MODULES` returns
a list of output modules available to the sender client.

    LIST OUTPUT_MODULES
    200-espeak-ng
    200-praat
    200 OK LIST OUTPUT_MODULES DONE

The `GET OUTPUT_MODULE` returns
the output module to be used
for speaking messages received from the sender client.

    GET OUTPUT_MODULE
    200-espeak-ng
    200 OK GET OUTPUT_MODULE DONE

The `SET <client> OUTPUT_MODULE <module>` command
changes the output module to be used
for speaking messagse of the sender client.
The `module` argument must be a value returned by `LIST OUTPUT_MODULES`.

    SET OUTPUT_MODULE espeak-ng
    200 OK SET OUTPUT_MODULE DONE

### Synthesis Voice

The `LIST SYNTHESIS_VOICES` command returns
synthesise voices that the current output module says is available.
For each voice, the voice name, language code and dialect are returned,
delimited by tabs.

    LIST SYNTHESIS_VOICES
    200-afaraf      aa  none
    200-tibetan     bo  none
    200-welsh       cy  none
    200-danish      da  none
    200-english     en  none
    200-french      fr  none
    200 OK LIST SYNTHESIS_VOICES DONE

The `SET <client> SYNTHESIS_VOICE <name>` command
changes the voice to use
for new messages received from the given client.
The `name` argument must be a value returned by `LIST SYNTHESIS_VOICES`.

    SET SELF SYNTHESIS_VOICE english
    200 OK SET SYNTHESIS_VOICE DONE
    SET SELF SYNTHESIS_VOICE clingon
    400 NO SET SYNTHESIS_VOICE UNKNOWN

### Voice Type

The `LIST VOICES` command returns common voice types
that can be used for speaking messages.

    LIST VOICES
    200-CHILD_FEMALE
    200-CHILD_MALE
    200-FEMALE1
    200-FEMALE2
    200-FEMALE3
    200-MALE1
    200-MALE2
    200-MALE3
    200 OK LIST VOICES DONE

The `GET VOICE_TYPE` command returns
the voice type to try to use
for new messages received from the sending client.

    GET VOICE_TYPE
    200-MALE1
    200 OK GET VOICE_TYPE DONE

The `SET <client> VOICE_TYPE <name>` command
changes the voice to use
for new messages received from the given client.

    SET SELF VOICE_TYPE MALE2
    200 OK SET VOICE_TYPE DONE
    SET SELF VOICE_TYPE CAT42
    400 NO SET VOICE_TYPE UNKNOWN

State Transitions
-----------------

These settings deal with when a message should transition
between speaking and non-speaking states
and what happens when they do.

### Notifications

The `SET SELF NOTIFICATION <type> <bool>` command
changes whether state changes of new messages from the sender client
should be sent to the client as asynchronous messages.
The `type` parameter determines what events should be sent.
If `type` is `all`, then all available event types are changed.

When an event is emitted,
a response of the following form is sent to the client.

    7XX-<msg-id>
    7XX-<client-id>
    [7XX-<index-mark>]
    7XX <event-type>

In it, the `msg-id` is the ID returned
from the response of a message request.
The `client-id` is that of the message sender.
The `index-mark` line is sent only for `INDEX_MARK` events
and contains the name of the index mark.
Lastly, the return code specifies the type emitted,
and `event-type` provides a implementation-defined
human-readable form of the event type.
The pairings of event types and return codes are as follows.

    700 INDEX_MARK
    701 BEGIN
    702 END
    703 CANCEL
    704 PAUSE
    705 RESUME

An example of commands and responses involving a notification
is as follows.

    SET SELF SSML_MODE ON
    200 OK SET SSML MODE DONE
    SET SELF NOTIFICATON ALL ON
    200 OK SET NOTIFICATION DONE
    SPEAK
    200 OK SPEAK AWAITING TEXT
    <speak>Hello, <mark name="hello"/> world!</speak>
    .
    200-10
    200 OK SPEAK QUEUED
    PAUSE SELF
    200 OK PAUSE DONE
    701-10
    701-1
    701 BEGIN
    700-10
    700-1
    700-hello
    700 INDEX_MARK
    704-10
    704-1
    704 PAUSE
    RESUME SELF
    200 OK RESUME DONE
    705-10
    705-1
    705 RESUME
    702-10
    702-1
    702 END

### Pause context

The `SET <target> PAUSE_CONTEXT <value>` command
changes the integer number `value` of sentences to repeat when resuming.

    SET SELF PAUSE_CONTEXT 1
    200 OK SET PAUSE_CONTEXT DONE

### Priority

The `SET SELF PRIORITY <priority>` command
changes the priority associated with
any new messages received from the sender client.

    SET SELF PRIORITY MESSAGE
    200 OK SET PRIORITY DONE

Blocks
------

The `BLOCK BEGIN` command starts a message block
that ends when when a `BLOCK END` command is received.
Messages that are sent after a block is started
are processed immediately as a message would be,
but it treated as part of the message block
for the purpose of history and scheduling.
Once a block is started, only the following commands are allowed.

-   Message request:  `CHAR`, `KEY`, `SOUND_ICON`, `SPEAK`.
-   Setting manipulation: `SET SELF <parameter> <...>`,
    where the `parameter` can be
    -   Processing option: `CAP_LET_RECOGN`, `PUNCTUATION`.
    -   Audio adjustments: `PITCH`, `RATE`, `VOLUME`.
    -   Voice settings: `LANGUAGE`, `SYNTHESIS_VOICE`, `VOICE_TYPE`.
-   Message grouping: `BLOCK END`
-   Client command: `QUIT`

In particular, the following are not possible inside a block.

-   Queue control: `CANCEL`, `PAUSE`, `RESUME`, `STOP`.
-   Setting manipulation: `SET <client> <...>`
    where the `client` is not `SELF` or the sender client ID.
-   Setting manipulation: `SET SELF <parameter> <...>`
    where the `parameter` can be
    (I am not sure all of these are intentionally disallowed)
    -   Processing option: `SPELLING`, `SSML_MODE`.
    -   Voice settings: `OUTPUT_MODULE`.
    -   State transitions: `NOTIFICTION`, `PAUSE_CONTEXT`, `PRIORITY`.
-   Message group: `BLOCK START`, meaning
    a message block cannot be started inside another message block.
-   Message query: `HISTORY`
-   Client commands: `QUIT`

The `BLOCK END` command stops a started message block.
This command allows more commands to be usable again.
This command can only be sent when a block is started.
(The specification does not mention what happens
if the block contains no messages.
That is, whether it is still considered a message,
or should just be ignored.)

    BLOCK BEGIN
    200 OK BLOCK BEGIN DONE
    KEY a
    200 OK KEY QUEUED
    KEY b
    200 OK KEY QUEUED
    BLOCK END
    200 OK BLOCK END DONE

Client Commands
---------------

Most commands targetting a client change settings for future messages.
These commands are concerned with the client itself.

### Client Name

The `SET SELF CLIENT_NAME <user>:<target>:<component>` command
changes the name of the sender client.

    SET SELF CLIENT_NAME alice:tail:new-messages
    200 OK SET CLIENT_NAME DONE
    SET SELF CLIENT_NAME alice:tail:new-messages
    400 NO SET CLIENT_NAME REPEATED

### History

The `SET <client> HISTORY <status>` command
changes whether to remember new messages received from the given client.

    SET SELF HISTORY OFF
    200 OK SET HISTORY DONE
    SET SELF HISTORY ON
    200 OK SET HISTORY DONE

### Help

The `HELP` command returns supported SSIP commands in a response.
(The specification does not say what else the returned message contains
nor how it should be formatted.)

    HELP
    200-BLOCK Message grouping
    200-CANCEL Stop current and queued messages
    200-CHAR Speak a character
    200-GET Query setting
    200-HELP Query available commands
    200-HISTORY Query received messages
    200-KEY Speak keys
    200-LIST Query setting values
    200-PAUSE Postpone current and queued messages
    200-QUIT Close connection
    200-RESUME Continue playing messages
    200-SET Change setting
    200-SOUND_ICON Play a predefined clip
    200-SPEAK Speak a text message
    200-STOP Stop current message
    200 OK HELP DONE

### Quit

The `QUIT` command closes the sender client connection.

    QUIT
    200 QUIT DONE

Server Commands
---------------

The `SET ALL DEBUG <bool>` command
determines whether to write debug information into a log file.
It is a server-wide setting.
If `bool` argument is `ON`,
the server responds with the log file name to be written into.

    SET ALL DEBUG ON
    200-/home/user/.local/state/speech-dispatcher/log/debug
    200 OK SET DEBUG DONE
    SET ALL DEBUG OFF
    200 OK SET DEBUG DONE

[speech dispatcher]: https://github.com/brailcom/speechd

[ssip]: https://github.com/brailcom/speechd/blob/master/doc/ssip.texi

[ssip webpage]: https://htmlpreview.github.io/?https://github.com/brailcom/speechd/blob/master/doc/ssip.html
