---
title: Android RemoteViews
date: 2019-08-19 23:14:05
tags: View
categories: Android
---

# Android RemoteViews

RemoteViews 远程视图，它不是用在应用自身的进程中，而是用在其他进程（SystemServer 进程）中显示视图界面，但它不是真正的 View，也没有继承自 View 类。RemoteViews 的使用场景有两种，通知栏和桌面小部件（Widget）。

## RemoteViews 在通知栏的应用

发送通知时可以使用 RemoteViews 完成自定义通知栏 View 的效果。

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.remote_views_activity);
        mNotificationManager = (NotificationManager) getApplication().getApplicationContext().getSystemService(Context.NOTIFICATION_SERVICE);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel("remote_views_chan", "remote views", NotificationManager.IMPORTANCE_HIGH);
            mNotificationManager.createNotificationChannel(channel);
        }
    }

    private Notification buildNotification() {
        Notification.Builder builder;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            builder = new Notification.Builder(this, "remote_views_chan");

        } else {
            builder = new Notification.Builder(this);
        }
        builder.setContentTitle("this is title")
                .setContentText("this is text")
                .setSmallIcon(R.mipmap.ic_launcher);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            RemoteViews contentView = new RemoteViews(getPackageName(), R.layout.notification_layout);
            contentView.setTextViewText(R.id.text, "this is remote view text");
            contentView.setImageViewResource(R.id.icon, R.mipmap.ic_launcher);
            builder.setCustomContentView(contentView);
        } else {
            builder.setContent(new RemoteViews(getPackageName(), R.layout.notification_layout));
        }
        return builder.build();
    }

    private void sendNotification() {
        mNotificationManager.notify(1, buildNotification());
    }
```

在 Android SDK 24 以上时，需要使用通知通道（ Notification Channel ）来发送通知。同时使用 `setCustomContentView` 来自定义通知栏视图。

和普通 View 不一样，RemoteViews 没有 findViewById 方法，它使用 `setTextViewText` 方法和 `setImageViewResource` 方法来更新文字和图片。
