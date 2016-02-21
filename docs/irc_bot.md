# irc.bot #

There are three classes in irc.bot:

ServerSpec: A simple data structure for keeping information about a server. In
particular, it's address, port, and a password to connect (if any).

SingleServerIRCBot: The proper 'bot' class of this library.  It inherits from the
client implementation in irc.client. Instead of using the server(), add_global_handler(),
remove_global_handler(), execute_at(), execute_delayed(), execute_every(), process_once(),
and process_forever() methods of that class, you use the start() method of this
class and handle different events by writing an on_<eventname> method for them.

Channel: A data structure for keeping information about a channel, supports a wide
variety of methods to query the data structure and answer questions such as "am I
oped in this channel?".

## SingleServerIRCBot ##

### Tutorial ###

Before we get started, it's important to understand a few things about how the
event system works. When you get a message from an IRC server over the wire, it
generally looks like this:

:<SERVER_HOSTNAME> 252 <NICK> 33 :operator(s) online

This kind of response is called a 'numeric reply'. Pay particular attention to
that three digit number between the hostname and the users nick, because that's
the numeric part and it tells you what kind of reply the message represents. In
the file events.py included in the source distribution there is a long list of
numeric reply codes and what event string they generate. Trying to decipher
what the codes mean from just the event strings can be a bit difficult, in rfc1459
(https://tools.ietf.org/html/rfc1459) section six there's a numeric reply code
dictionary explaining what they all mean.

With that in mind, there's a working sample bot in the file scripts/testbot.py
included with the source distribution. The rest of this tutorial will just be
an analysis of this file. The exact code in the sample bot may differ a bit from
this file, but they should look about the same:

#### Imports ####

    import irc.bot
    import irc.strings
    from irc.client import ip_numstr_to_quad, ip_quad_to_numstr

This should be fairly straightforward, however as a note the third import is used
as part of handling DCC connections, if you don't need DCC then you can ignore it.

#### Instantiation ####

#### Commands ####
