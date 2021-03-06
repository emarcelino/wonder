What is it
**************************************

ERIMAdaptor provides an Instant Messenger interface to your WOComponents. There are several limitations that, however,
 you need to be aware of.  Only one IMAction per-page can be triggered (the last one that is visible to the user, specifically).

The architecture of this adaptor is someone different than the normal HTTP adaptor, because of inherent limitations in the
IM interfaces like AOL IM:

* WOSessions are tracked by buddy name.  When a buddy contacts the server's IM account, a Conversation is initiated.  A
Conversation associates a buddy name with the WOSession ID.  Conversations have a configurable expiration, but the default
is to expire after 5 minutes of inactivity.  The WOSession will have the same expiration as they normally would, but the 
connection between the buddy name and that session will go away, effectively expiring the session's usefulness.

* Different IM networks may impose additional restrictions on your components.  For instance, most networks limit the 
number of bytes that can be transferred in a message.  Currently the framework doesn't auto-split, but that's an 
obvious enhancement that will probably be added in.

* Some features of WORequest and WOResponse may not behave exactly like the HTTP counterpart.  For instance, there are
no normal HTTP headers in the WORequest.

* The buddy name and message appear as form values and in the request userInfo dictionary -- use whichever is more convenient.

* The Conversation object is accessible via the request userInfo dictionary if you need direct access (to force expiration, etc).

* The implementation is TEMPORARILY single threaded for all conversations.  This will change soon, I just haven't had
time to test it.

Currently there are implementions of the IInstantMessengerFactory for AIM using the jaimbot, daim, and joscar libraries.  Jaimbot
is available from jaimbot.sourceforge.net and daim is available from daim.dev.java.net if you're interested in getting
more information on either library.  JOscar, which seems to be the most capable of the three, is available at http://joust.kano.net/.

How to use it
**************************************

ERIMAdaptor is a custom adaptor, so it can be added using the WOAdditionalAdaptors option on your WO:

-WOAdditionalAdaptors ({WOAdaptor="er.imadaptor.InstantMessengerAdaptor";})


There are several new settings that can (or must) appear in your Properties file as well:

IMFactory (optional, default er.imadaptor.AimBotInstantMessenger$Factory) - the factory class for creating IM connections.  If you
	want to provide a new type of IM network, you should implement the IInstantMessengerFactory interface for whatever class you 
	provide here.
IMScreenName (required) - the screen name of the IM account that the server should login with
IMPassword (required) - the password of the IM account that the server should login with
IMTimeout (optional, default 5 minutes) - the conversation timeout time in milliseconds
IMAutoLogin (optional, default "true") - whether or not the adaptor should autologin to AIM.  If false, you must call adaptor.connect()
IMConversationActionName (optional, default "imConversation") - the name of the DirectAction to call when a Conversation 
	is initiated, the default is that you must create a public WOActionResults imConversationAction() { .. } method in your
	DirectAction class
IMWatcherEnabled (optional, default "false") - whether or not you want to have a second AIM account login and
	watchdog the first.  Most of the AIM libraries can have issues with getting kicked off after a period of time.  If you
	have two AIM accounts, they can keep eachother alive.
IMWatcherFactory (optional, default same as IMFactory) - if IMWatcherEnabled, the factory class to use
IMWatcherScreenName (optional, required if IMWatcherEnabled) - the screen name of the watcher IM account
IMWatcherPassword (optional, required if IMWatcherEnabled) - the password of the watcher IM account

In your code, the following request headers are available:

IsIM - Boolean.TRUE if the current request is an IM request (you can call InstantMessengerAdaptor.isIMRequest(WORequest) also)
IMConversation - the Conversation associated with this request
BuddyName - the name of the buddy that initiated the request
Message - the message sent by the user


The following form values are available in the request:

BuddyName - the name of the buddy that initiated the request
Message - the message sent by the user


For each request, the IM Adaptor needs to know what action to call when the subsequent request comes in.  In a normal WOComponent,
think of this as each page having a "Next" button on it that is clicked each time a request comes in.  Instead of using a 
WOSubmitButton or a WOHyperlink, you instead use the IMAction element.  This element has one attribute -- "action" that points to
the action method to call on your component.  So for instance:

.java:
public class IMComponent extends WOComponent {
	public boolean userResponded;
	
	...

  public String buddyName() {
    return InstantMessengerAdaptor.buddyName(context().request());
  }

	public WOActionResults processResponse() {
		userResponded = true;
		return null;
	}
}

.html:
<webobject name = "FirstContactConditional">
	Hi <webobject name = "BuddyName"></webobject>!
</webobject>
<webobject name = "UserRespondedConditional">
	Welcome back <webobject name = "BuddyName"></webobject>!
</webobject>
<webobject name = "IMAction"></webobject>

.wod:
FirstContactConditional : WOConditional {
	condition = userResponded;
	negate = true;
}

UserRespondedConditional : WOConditional {
	condition = userResponded;
}

BuddyName : WOString {
	value = buddyName;
}

IMAction : IMAction {
	action = processResponse;
}

Transcript:
Mike> Hey server!
Server> Hi Mike!
Mike> How's it going?
Server> Welcome back Mike!
Mike> Uh. OK.
Server> Welcome back Mike!
	...

