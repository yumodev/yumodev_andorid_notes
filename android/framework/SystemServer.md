# SystemServer

[SystemServer.java](https://www.androidos.net.cn/android/2.3.7_r1/xref/frameworks/base/services/java/com/android/server/SystemServer.java)
## 概述

SystemServer进程非常重要，几乎所有的核心服务都在这个进程中，比如AMS，PMS，WMS等等。

## SystemServer 分析

SystemServer是由Zygote孵化二来的一个进程，名字为system_server.

其main函数如下：


```java
 public static void main(String[] args) {
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            // If a device's clock is before 1970 (before 0), a lot of
            // APIs crash dealing with negative numbers, notably
            // java.io.File#setLastModified, so instead we fake it and
            // hope that time from cell towers or NTP fixes it
            // shortly.
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }

        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            timer = new Timer();
            timer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server");
                }
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }

        // The system server has to run all of the time, so it needs to be
        // as efficient as possible with its memory usage.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
        
        System.loadLibrary("android_servers");
        init1(args);// native方法
    }
    
   native public static void init1(String[] args);
```

init1是native方法，其中又调用了SystemServer的init2方法

## 初始化Service

```java

 public static final void init2() {
    Slog.i(TAG, "Entered the Android system server!");
    Thread thr = new ServerThread();
    thr.setName("android.server.ServerThread");
    thr.start();
}


class ServerThread extends Thread {
    private static final String TAG = "SystemServer";
    private final static boolean INCLUDE_DEMO = false;

    private static final int LOG_BOOT_PROGRESS_SYSTEM_RUN = 3010;

    private ContentResolver mContentResolver;

    private class AdbSettingsObserver extends ContentObserver {
        public AdbSettingsObserver() {
            super(null);
        }
        @Override
        public void onChange(boolean selfChange) {
            boolean enableAdb = (Settings.Secure.getInt(mContentResolver,
                Settings.Secure.ADB_ENABLED, 0) > 0);
            // setting this secure property will start or stop adbd
           SystemProperties.set("persist.service.adb.enable", enableAdb ? "1" : "0");
        }
    }

    @Override
    public void run() {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN,
            SystemClock.uptimeMillis());

        //开启looper。
        Looper.prepare();
        //设置线程优先级
        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);

        BinderInternal.disableBackgroundScheduling(true);
        android.os.Process.setCanSelfBackground(false);

        // Check whether we failed to shut down last time we tried.
        {
            final String shutdownAction = SystemProperties.get(
                    ShutdownThread.SHUTDOWN_ACTION_PROPERTY, "");
            if (shutdownAction != null && shutdownAction.length() > 0) {
                boolean reboot = (shutdownAction.charAt(0) == '1');

                final String reason;
                if (shutdownAction.length() > 1) {
                    reason = shutdownAction.substring(1, shutdownAction.length());
                } else {
                    reason = null;
                }

                ShutdownThread.rebootOrShutdown(reboot, reason);
            }
        }

        String factoryTestStr = SystemProperties.get("ro.factorytest");
        int factoryTest = "".equals(factoryTestStr) ? SystemServer.FACTORY_TEST_OFF
                : Integer.parseInt(factoryTestStr);

        LightsService lights = null;
        PowerManagerService power = null;
        BatteryService battery = null;
        ConnectivityService connectivity = null;
        IPackageManager pm = null;
        Context context = null;
        WindowManagerService wm = null;
        BluetoothService bluetooth = null;
        BluetoothA2dpService bluetoothA2dp = null;
        HeadsetObserver headset = null;
        DockObserver dock = null;
        UsbService usb = null;
        UiModeManagerService uiMode = null;
        RecognitionManagerService recognition = null;
        ThrottleService throttle = null;

        // Critical services...
        try {
            //熵服务和随机数有关
            Slog.i(TAG, "Entropy Service");
            ServiceManager.addService("entropy", new EntropyService());

            //电力服务
            Slog.i(TAG, "Power Manager");
            power = new PowerManagerService();
            ServiceManager.addService(Context.POWER_SERVICE, power);

            Slog.i(TAG, "Activity Manager");
            //调用AMS服务的main方法
            context = ActivityManagerService.main(factoryTest);

            //电话面试
            Slog.i(TAG, "Telephony Registry");
            ServiceManager.addService("telephony.registry", new TelephonyRegistry(context));

            AttributeCache.init(context);

            Slog.i(TAG, "Package Manager");
            pm = PackageManagerService.main(context,
                    factoryTest != SystemServer.FACTORY_TEST_OFF);

            //设置系统进程
            ActivityManagerService.setSystemProcess();

            mContentResolver = context.getContentResolver();

            // The AccountManager must come before the ContentService
            try {
                Slog.i(TAG, "Account Manager");
                ServiceManager.addService(Context.ACCOUNT_SERVICE,
                        new AccountManagerService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Account Manager", e);
            }

            Slog.i(TAG, "Content Manager");
            ContentService.main(context,
                    factoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL);

            Slog.i(TAG, "System Content Providers");
            ActivityManagerService.installSystemProviders();

            Slog.i(TAG, "Battery Service");
            battery = new BatteryService(context);
            ServiceManager.addService("battery", battery);

            Slog.i(TAG, "Lights Service");
            lights = new LightsService(context);

            Slog.i(TAG, "Vibrator Service");
            ServiceManager.addService("vibrator", new VibratorService(context));

            // only initialize the power service after we have started the
            // lights service, content providers and the battery service.
            power.init(context, lights, ActivityManagerService.getDefault(), battery);

            Slog.i(TAG, "Alarm Manager");
            AlarmManagerService alarm = new AlarmManagerService(context);
            ServiceManager.addService(Context.ALARM_SERVICE, alarm);

            Slog.i(TAG, "Init Watchdog");
            Watchdog.getInstance().init(context, battery, power, alarm,
                    ActivityManagerService.self());

            Slog.i(TAG, "Window Manager");
            wm = WindowManagerService.main(context, power,
                    factoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL);
            ServiceManager.addService(Context.WINDOW_SERVICE, wm);

            ((ActivityManagerService)ServiceManager.getService("activity"))
                    .setWindowManager(wm);

            // Skip Bluetooth if we have an emulator kernel
            // TODO: Use a more reliable check to see if this product should
            // support Bluetooth - see bug 988521
            if (SystemProperties.get("ro.kernel.qemu").equals("1")) {
                Slog.i(TAG, "Registering null Bluetooth Service (emulator)");
                ServiceManager.addService(BluetoothAdapter.BLUETOOTH_SERVICE, null);
            } else if (factoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL) {
                Slog.i(TAG, "Registering null Bluetooth Service (factory test)");
                ServiceManager.addService(BluetoothAdapter.BLUETOOTH_SERVICE, null);
            } else {
                Slog.i(TAG, "Bluetooth Service");
                bluetooth = new BluetoothService(context);
                ServiceManager.addService(BluetoothAdapter.BLUETOOTH_SERVICE, bluetooth);
                bluetooth.initAfterRegistration();
                bluetoothA2dp = new BluetoothA2dpService(context, bluetooth);
                ServiceManager.addService(BluetoothA2dpService.BLUETOOTH_A2DP_SERVICE,
                                          bluetoothA2dp);

                int bluetoothOn = Settings.Secure.getInt(mContentResolver,
                    Settings.Secure.BLUETOOTH_ON, 0);
                if (bluetoothOn > 0) {
                    bluetooth.enable();
                }
            }

        } catch (RuntimeException e) {
            Slog.e("System", "Failure starting core service", e);
        }

        DevicePolicyManagerService devicePolicy = null;
        StatusBarManagerService statusBar = null;
        InputMethodManagerService imm = null;
        AppWidgetService appWidget = null;
        NotificationManagerService notification = null;
        WallpaperManagerService wallpaper = null;
        LocationManagerService location = null;

        if (factoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
            try {
                Slog.i(TAG, "Device Policy");
                devicePolicy = new DevicePolicyManagerService(context);
                ServiceManager.addService(Context.DEVICE_POLICY_SERVICE, devicePolicy);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting DevicePolicyService", e);
            }

            try {
                Slog.i(TAG, "Status Bar");
                statusBar = new StatusBarManagerService(context);
                ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting StatusBarManagerService", e);
            }

            try {
                Slog.i(TAG, "Clipboard Service");
                ServiceManager.addService(Context.CLIPBOARD_SERVICE,
                        new ClipboardService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Clipboard Service", e);
            }

            try {
                Slog.i(TAG, "Input Method Service");
                imm = new InputMethodManagerService(context, statusBar);
                ServiceManager.addService(Context.INPUT_METHOD_SERVICE, imm);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Input Manager Service", e);
            }

            try {
                Slog.i(TAG, "NetStat Service");
                ServiceManager.addService("netstat", new NetStatService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting NetStat Service", e);
            }

            try {
                Slog.i(TAG, "NetworkManagement Service");
                ServiceManager.addService(
                        Context.NETWORKMANAGEMENT_SERVICE,
                        NetworkManagementService.create(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting NetworkManagement Service", e);
            }

            try {
                Slog.i(TAG, "Connectivity Service");
                connectivity = ConnectivityService.getInstance(context);
                ServiceManager.addService(Context.CONNECTIVITY_SERVICE, connectivity);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Connectivity Service", e);
            }

            try {
                Slog.i(TAG, "Throttle Service");
                throttle = new ThrottleService(context);
                ServiceManager.addService(
                        Context.THROTTLE_SERVICE, throttle);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting ThrottleService", e);
            }

            try {
              Slog.i(TAG, "Accessibility Manager");
              ServiceManager.addService(Context.ACCESSIBILITY_SERVICE,
                      new AccessibilityManagerService(context));
            } catch (Throwable e) {
              Slog.e(TAG, "Failure starting Accessibility Manager", e);
            }

            try {
                /*
                 * NotificationManagerService is dependant on MountService,
                 * (for media / usb notifications) so we must start MountService first.
                 */
                Slog.i(TAG, "Mount Service");
                ServiceManager.addService("mount", new MountService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Mount Service", e);
            }

            try {
                Slog.i(TAG, "Notification Manager");
                notification = new NotificationManagerService(context, statusBar, lights);
                ServiceManager.addService(Context.NOTIFICATION_SERVICE, notification);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Notification Manager", e);
            }

            try {
                Slog.i(TAG, "Device Storage Monitor");
                ServiceManager.addService(DeviceStorageMonitorService.SERVICE,
                        new DeviceStorageMonitorService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting DeviceStorageMonitor service", e);
            }

            try {
                Slog.i(TAG, "Location Manager");
                location = new LocationManagerService(context);
                ServiceManager.addService(Context.LOCATION_SERVICE, location);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Location Manager", e);
            }

            try {
                Slog.i(TAG, "Search Service");
                ServiceManager.addService(Context.SEARCH_SERVICE,
                        new SearchManagerService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Search Service", e);
            }

            if (INCLUDE_DEMO) {
                Slog.i(TAG, "Installing demo data...");
                (new DemoThread(context)).start();
            }

            try {
                Slog.i(TAG, "DropBox Service");
                ServiceManager.addService(Context.DROPBOX_SERVICE,
                        new DropBoxManagerService(context, new File("/data/system/dropbox")));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting DropBoxManagerService", e);
            }

            try {
                Slog.i(TAG, "Wallpaper Service");
                wallpaper = new WallpaperManagerService(context);
                ServiceManager.addService(Context.WALLPAPER_SERVICE, wallpaper);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Wallpaper Service", e);
            }

            try {
                Slog.i(TAG, "Audio Service");
                ServiceManager.addService(Context.AUDIO_SERVICE, new AudioService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Audio Service", e);
            }

            try {
                Slog.i(TAG, "Headset Observer");
                // Listen for wired headset changes
                headset = new HeadsetObserver(context);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting HeadsetObserver", e);
            }

            try {
                Slog.i(TAG, "Dock Observer");
                // Listen for dock station changes
                dock = new DockObserver(context, power);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting DockObserver", e);
            }

            try {
                Slog.i(TAG, "USB Service");
                // Listen for USB changes
                usb = new UsbService(context);
                ServiceManager.addService(Context.USB_SERVICE, usb);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting UsbService", e);
            }

            try {
                Slog.i(TAG, "UI Mode Manager Service");
                // Listen for UI mode changes
                uiMode = new UiModeManagerService(context);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting UiModeManagerService", e);
            }

            try {
                Slog.i(TAG, "Backup Service");
                ServiceManager.addService(Context.BACKUP_SERVICE,
                        new BackupManagerService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Backup Service", e);
            }

            try {
                Slog.i(TAG, "AppWidget Service");
                appWidget = new AppWidgetService(context);
                ServiceManager.addService(Context.APPWIDGET_SERVICE, appWidget);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting AppWidget Service", e);
            }

            try {
                Slog.i(TAG, "Recognition Service");
                recognition = new RecognitionManagerService(context);
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting Recognition Service", e);
            }
            
            try {
                Slog.i(TAG, "DiskStats Service");
                ServiceManager.addService("diskstats", new DiskStatsService(context));
            } catch (Throwable e) {
                Slog.e(TAG, "Failure starting DiskStats Service", e);
            }
        }

        // make sure the ADB_ENABLED setting value matches the secure property value
        Settings.Secure.putInt(mContentResolver, Settings.Secure.ADB_ENABLED,
                "1".equals(SystemProperties.get("persist.service.adb.enable")) ? 1 : 0);

        // register observer to listen for settings changes
        mContentResolver.registerContentObserver(Settings.Secure.getUriFor(Settings.Secure.ADB_ENABLED),
                false, new AdbSettingsObserver());

        // Before things start rolling, be sure we have decided whether
        // we are in safe mode.
        final boolean safeMode = wm.detectSafeMode();
        if (safeMode) {
            try {
                ActivityManagerNative.getDefault().enterSafeMode();
                // Post the safe mode state in the Zygote class
                Zygote.systemInSafeMode = true;
                // Disable the JIT for the system_server process
                VMRuntime.getRuntime().disableJitCompilation();
            } catch (RemoteException e) {
            }
        } else {
            // Enable the JIT for the system_server process
            VMRuntime.getRuntime().startJitCompilation();
        }

        // It is now time to start up the app processes...

        if (devicePolicy != null) {
            devicePolicy.systemReady();
        }

        if (notification != null) {
            notification.systemReady();
        }

        if (statusBar != null) {
            statusBar.systemReady();
        }
        wm.systemReady();
        power.systemReady();
        try {
            pm.systemReady();
        } catch (RemoteException e) {
        }

        // These are needed to propagate to the runnable below.
        final StatusBarManagerService statusBarF = statusBar;
        final BatteryService batteryF = battery;
        final ConnectivityService connectivityF = connectivity;
        final DockObserver dockF = dock;
        final UsbService usbF = usb;
        final ThrottleService throttleF = throttle;
        final UiModeManagerService uiModeF = uiMode;
        final AppWidgetService appWidgetF = appWidget;
        final WallpaperManagerService wallpaperF = wallpaper;
        final InputMethodManagerService immF = imm;
        final RecognitionManagerService recognitionF = recognition;
        final LocationManagerService locationF = location;

        // We now tell the activity manager it is okay to run third party
        // code.  It will call back into us once it has gotten to the state
        // where third party code can really run (but before it has actually
        // started launching the initial applications), for us to complete our
        // initialization.
        ((ActivityManagerService)ActivityManagerNative.getDefault())
                .systemReady(new Runnable() {
            public void run() {
                Slog.i(TAG, "Making services ready");

                if (statusBarF != null) statusBarF.systemReady2();
                if (batteryF != null) batteryF.systemReady();
                if (connectivityF != null) connectivityF.systemReady();
                if (dockF != null) dockF.systemReady();
                if (usbF != null) usbF.systemReady();
                if (uiModeF != null) uiModeF.systemReady();
                if (recognitionF != null) recognitionF.systemReady();
                Watchdog.getInstance().start();

                // It is now okay to let the various system services start their
                // third party code...

                if (appWidgetF != null) appWidgetF.systemReady(safeMode);
                if (wallpaperF != null) wallpaperF.systemReady();
                if (immF != null) immF.systemReady();
                if (locationF != null) locationF.systemReady();
                if (throttleF != null) throttleF.systemReady();
            }
        });

        // For debug builds, log event loop stalls to dropbox for analysis.
        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        Looper.loop();
        Slog.d(TAG, "System ServerThread is exiting!");
    }
}

```
