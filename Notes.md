https://stackoverflow.com/questions/44425584/context-startforegroundservice-did-not-then-call-service-startforeground


6

I have a work around for this problem. I have verified this fix in my own app(300K+ DAU), which can reduce at least 95% of this kind of crash, but still cannot 100% avoid this problem.

This problem happens even when you ensure to call startForeground() just after service started as Google documented. It may be because the service creation and initialization process already cost more than 5 seconds in many scenarios, then no matter when and where you call startForeground() method, this crash is unavoidable.

My solution is to ensure that startForeground() will be executed within 5 seconds after startForegroundService() method, no matter how long your service need to be created and initialized. Here is the detailed solution.

Do not use startForegroundService at the first place, use bindService() with auto_create flag. It will wait for the service initialization. Here is the code, my sample service is MusicService:

final Context applicationContext = context.getApplicationContext();
Intent intent = new Intent(context, MusicService.class);
applicationContext.bindService(intent, new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder binder) {
        if (binder instanceof MusicBinder) {
            MusicBinder musicBinder = (MusicBinder) binder;
            MusicService service = musicBinder.getService();
            if (service != null) {
                // start a command such as music play or pause.
                service.startCommand(command);
                // force the service to run in foreground here.
                // the service is already initialized when bind and auto_create.
                service.forceForeground();
            }
        }
        applicationContext.unbindService(this);
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
    }
}, Context.BIND_AUTO_CREATE);
Then here is MusicBinder implementation:

/**
 * Use weak reference to avoid binder service leak.
 */
 public class MusicBinder extends Binder {

     private WeakReference<MusicService> weakService;

     /**
      * Inject service instance to weak reference.
      */
     public void onBind(MusicService service) {
         this.weakService = new WeakReference<>(service);
     }

     public MusicService getService() {
         return weakService == null ? null : weakService.get();
     }
 }
The most important part, MusicService implementation, forceForeground() method will ensure that startForeground() method is called just after startForegroundService():

public class MusicService extends MediaBrowserServiceCompat {
...
    private final MusicBinder musicBind = new MusicBinder();
...
    @Override
    public IBinder onBind(Intent intent) {
        musicBind.onBind(this);
        return musicBind;
    }
...
    public void forceForeground() {
        // API lower than 26 do not need this work around.
        if (Build.VERSION.SDK_INT >= 26) {
            Intent intent = new Intent(this, MusicService.class);
            // service has already been initialized.
            // startForeground method should be called within 5 seconds.
            ContextCompat.startForegroundService(this, intent);
            Notification notification = mNotificationHandler.createNotification(this);
            // call startForeground just after startForegroundService.
            startForeground(Constants.NOTIFICATION_ID, notification);
        }
    }
}
If you want to run the step 1 code snippet in a pending intent, such as if you want to start a foreground service in a widget (a click on widget button) without opening your app, you can wrap the code snippet in a broadcast receiver, and fire a broadcast event instead of start service command.

That is all. Hope it helps. Good luck.



-----------



https://stackoverflow.com/questions/46803663/context-startforegroundservice-did-not-then-call-service-startforeground


I experienced this issue for Galaxy Tab A(Android 8.1.0). In my application, I start a foreground service in the constructor of MainApplication(Application-extended class). With the Debugger, I found it took more than 5 second to reach OnCreate() of the service after startForegroundService(intent). I got crash even though startForeground(ON_SERVICE_CONNECTION_NID, notification) got called in OnCreate().

After trying different approach, I found one works for me as the following:

In the constructor, I used AlarmManager to wakeup a receiver, and in the receiver, I started the foreground service.

I guessed that the reason I had the issue because heavy workload of application starting delayed the creation of the foreground service.

Try my approach, you might resolve the issue as well. Good luck.


------------


https://stackoverflow.com/questions/55894636/android-9-pie-context-startforegroundservice-did-not-then-call-service-star
