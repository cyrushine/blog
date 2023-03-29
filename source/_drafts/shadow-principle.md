# Application



# Activity/View

plugin.apk 里的 Activity 并没有注册在 host.apk 的清单文件里，所以 plugin Activity 并不是真正的 Android component，所以：

1. plugin view 没法通过 Activity 显示出来，除非用 `WindowManager.addView()`，但这是需要悬浮窗权限的，很显然 app 一般不可能拿到这个权限
2. 即使通过 `WindowManager.addView()` 显示出来，也无法收到 Activity 生命周期回调：`onResume`、`onPause`、`onDestroy`...
3. 无法处理涉及 Activity API 的相关调用
4. ...

于是需要用一个 `桩` 作为所有 plugin activity 的壳，这里统一称为 `shell activity`（`PluginDefaultProxyActivity`）

shell activity 实例存在于 plugin process 并注册在 host.apk 的清单文件里，它是一个正常的 Android component，plugin activity 实例存在于 plugin process，它只是一个普通的 java 对象

## lifecycle event

shell activity 是个正常的 Android component 所以会有生命周期事件，它将其转发给 plugin activity，这样 plugin 业务就可以正常执行

```java
public class PluginDefaultProxyActivity extends PluginContainerActivity

public class PluginContainerActivity extends GeneratedPluginContainerActivity implements HostActivity, HostActivityDelegator

abstract class GeneratedPluginContainerActivity extends Activity implements GeneratedHostActivityDelegator {

  GeneratedHostActivityDelegate hostActivityDelegate;

  @Override
  public void onCreate(Bundle arg0, PersistableBundle arg1) {
    if (hostActivityDelegate != null) {
      hostActivityDelegate.onCreate(arg0, arg1);
    } else {
      super.onCreate(arg0, arg1);
    }
  }

  @Override
  protected void onStart() {
    if (hostActivityDelegate != null) {
      hostActivityDelegate.onStart();
    } else {
      super.onStart();
    }
  }

  // onResume()
  // onPause()
  // onStop()
  // onDestroy()
  // onNewIntent(Intent)
  // onConfigurationChanged(Configuration)
  // ...  
}
```

## ShadowActivity

plugin activity 不是真正的 Android component，所以它的一些方法如 `Activity.setContentView(layoutRes)` 是不生效的，需要被 shadow gradle plugin 改造为继承自 `ShadowActivity`

```java
public class ShadowActivity extends PluginActivity {

    // 将 layout resource 设置到 shell activity 上才能在屏幕上显示出来
    @Override
    public void setContentView(int layoutResID) {
        if ("merge".equals(XmlPullParserUtil.getLayoutStartTagName(getResources(), layoutResID))) {
            //如果传进来的xml文件的根tag是merge时，需要特殊处理
            View decorView = hostActivityDelegator.getWindow().getDecorView();
            ViewGroup viewGroup = decorView.findViewById(android.R.id.content);
            LayoutInflater.from(this).inflate(layoutResID, viewGroup);
        } else {
            View inflate = LayoutInflater.from(this).inflate(layoutResID, null);
            hostActivityDelegator.setContentView(inflate);
        }
    }

    // plugin process 实际上是 host process 的子进程，所以 Application 是 HostApplication
    // 这里得返回 PluginApplication 才能逻辑完备
    @Override
    public final ShadowApplication getApplication() {
        return mPluginApplication;
    }


    @Override
    public void startActivityForResult(Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }

    @Override
    public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        final Intent pluginIntent = new Intent(intent);
        pluginIntent.setExtrasClassLoader(mPluginClassLoader);
        ComponentName callingActivity = new ComponentName(getPackageName(), getClass().getName());
        // true 表示该 Intent 是为了启动插件内 Activity 的，已经被正确消费了
        // false 表示该 Intent 不是插件内的 Activity
        final boolean success = mPluginComponentLauncher.startActivityForResult(hostActivityDelegator, pluginIntent, requestCode, options, callingActivity);
        if (!success) {
            hostActivityDelegator.startActivityForResult(intent, requestCode, options);
        }
    }
}
```

## 



# ContentProvider








