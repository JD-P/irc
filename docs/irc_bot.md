# irc.bot #

There are three classes in irc.bot:

ServerSpec: A simple data structure for keeping information about a server. In
particular, it's address, port, and a password to connect (if any).

SingleServerIRCBot: The proper 'bot' class of this library.  It inherits from the
client implementation in irc.client. Instead of using the server(), add_global_handler(),
remove_global_handler(), execute_at(), execute_delayed(), execute_every(), process_once(),
and process_forever() methods of that class, you can use the start() method of this
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

    class TestBot(irc.bot.SingleServerIRCBot):
        def __init__(self, channel, nickname, server, port=6667):
            irc.bot.SingleServerIRCBot.__init__(self, [(server, port)], nickname, nickname)
            self.channel = channel

We start by calling the .__init__() for the parent class irc.bot.SingleServerIRCBot.
If you have previous python experience this should also be fairly straightforward,
self.channel is later used as the initial channel to join to on connect.

#### Event Handlers ####

Next up we start seeing methods like this:

    def on_nicknameinuse(self, c, e):
        c.nick(c.get_nickname() + "_")

The cryptic 'c' and 'e' paramaters might have you scratching your head. They're
shorthand for 'connection' and 'event' respectively. Connection is the
irc.client.ServerConnection object associated with the bot, which we call methods
on to interact with the IRC server. Here we send a nick change to the server by
calling the nick() method with our current nickname from the get_nickname()
method and append an underscore to it. The event parameter goes unused in this
particular response but both the connection and event paramaters are mandatory
for event handlers.

    def on_privmsg(self, c, e):
        self.do_command(e, e.arguments[0])

This one by contrast makes use of the event parameter. The argument given to event
is a data structure of the form:

    type: <MESSAGE_TYPE>,
    source: <USER/HOST THE EVENT ORIGINATES FROM>,
    target: <USER/CHANNEL THE MESSAGE IS GOING TO>,
    arguments: <ACCOMPANYING INFORMATION IN A LIST>,
    tags: <TAGS IN A LIST>

Here's an example:

    type: privmsg,
    source: nick!irssi@192.168.1.1,
    target: test,
    arguments: ['Hi.'],
    tags: []

Which represents a privmsg. It should be noted that unlike the rfc, here a
privmsg means an actual private message sent to an individual user, not a channel,
messages sent to a channel are of the type pubmsg. The privmsg is sent by nick
to a user named test with the message 'Hi.'. There are no tags.

#### Commands ####

    def do_command(self, e, cmd):
        nick = e.source.nick
        c = self.connection

        if cmd == "disconnect":
            self.disconnect()
        elif cmd == "die":
            self.die()
        elif cmd == "stats":
            for chname, chobj in self.channels.items():
                c.notice(nick, "--- Channel statistics ---")
                c.notice(nick, "Channel: " + chname)
                users = sorted(chobj.users())
                c.notice(nick, "Users: " + ", ".join(users))
                opers = sorted(chobj.opers())
                c.notice(nick, "Opers: " + ", ".join(opers))
                voiced = sorted(chobj.voiced())
                c.notice(nick, "Voiced: " + ", ".join(voiced))
        elif cmd == "dcc":
            dcc = self.dcc_listen()
            c.ctcp("DCC", nick, "CHAT chat %s %d" % (
                ip_quad_to_numstr(dcc.localaddress),
                dcc.localport))
        else:
            c.notice(nick, "Not understood: " + cmd)

Now we come to the meat of the bot. There's quite a bit going on here so let's
break it down starting with the function header. do_command() takes two paramaters,
the first, e, is the standard event data structure we've already discussed. The
second, cmd, is the zeroth item in the arguments attribute of e. The do_command()
method performs different functions depending on what 'command' has been sent to
the bot. The zeroth argument of e is a string that is compared against different
commands to determine which, if any, the bot has been asked to do. Once a branch
has been decided on in the if conditional the command is executed.

#### Starting The Bot ####

def main():
    import sys
    if len(sys.argv) != 4:
        print(sys.argv)
        print("Usage: testbot <server[:port]> <channel> <nickname>")
        sys.exit(1)

    s = sys.argv[1].split(":", 1)
    server = s[0]
    if len(s) == 2:
        try:
            port = int(s[1])
        except ValueError:
            print("Error: Erroneous port.")
            sys.exit(1)
    else:
        port = 6667
    channel = sys.argv[2]
    nickname = sys.argv[3]

    bot = TestBot(channel, nickname, server, port)
    bot.start()

Here we have some fairly straightforward setup. The bot takes its initial
parameters such as what server to connect to from sys.argv. This sort of manual
parsing works but for a 'real' program I would recommend a higher level library
such as argparse.

Finally we create an instance of the bot and call its start() method to get it
running. Keep in mind that the event loop is set up so that the program will find
its 'entry point' on the welcome event generated by connecting to the server.

I had some difficulty getting the bot to run. On my system I had to replace the
shebang line pointing to 'python' to 'python3', then the bot ran from the scripts
directory with this:

    ./testbot.py irc.freenode.org:6667 '#bot' bot_test

On Linux.