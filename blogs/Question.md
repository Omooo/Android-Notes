```java


    /**
     * 我想统计两个类中方法执行耗时，BootActivity 中的统计没什么问题，
     * 但是 CommonRequest 类中的统计直接导致应用崩溃
     * 类转换异常？很萌
     * AspectJ 才学，希望大佬能帮帮我，不胜感激
     */

com.xxx.BootActivity
public class BootActivity extends Activity{
	onCreate(){
		CommonRequest.method1(this);
		CommonRequest.method2(this);
		//...
	}
}

com.yyy.CommonRequest
public class CommonRequest{
	public static void mothod1(Context context){
		//网络请求
		new HttpRequest(context).callback(new CallBack()){
			//
		}
	}
}

//切面类
@Aspect
public class MethodTimeTracker {

	//这个切点执行正常
    @Around("execution( * com.xxx.BootActivity.* (..))")
    public void bootTracker(ProceedingJoinPoint joinPoint) throws Throwable {
        long beginTime = SystemClock.currentThreadTimeMillis();
        joinPoint.proceed();
        long endTime = SystemClock.currentThreadTimeMillis();
        long dx = endTime - beginTime;
        Log.i("2333", joinPoint.getSignature().getDeclaringType().getName() + "#" + joinPoint.getSignature().getName()
                + " " + dx + " ms");
    }

    //这个切点直接导致应用崩溃
    @Around("execution( * com.yyy.CommonRequest.* (..))")
    public void requestTracker(ProceedingJoinPoint joinPoint) throws Throwable {
        long beginTime = SystemClock.currentThreadTimeMillis();
        joinPoint.proceed();
        long endTime = SystemClock.currentThreadTimeMillis();
        long dx = endTime - beginTime;
        Log.i("2333", joinPoint.getSignature().getDeclaringType().getName() + "#" + joinPoint.getSignature().getName()
                + " " + dx + " ms");
    }
}

Log 日志：
    java.lang.RuntimeException: Unable to start activity ComponentInfo{.BootActivity}: java.lang.ClassCastException: BootActivity cannot be cast to cn.shuhe.dmnetwork.network.CjjHttpRequest
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2913)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3048)
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1808)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loop(Looper.java:193)
        at android.app.ActivityThread.main(ActivityThread.java:6669)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
     Caused by: java.lang.ClassCastException: BootActivity cannot be cast to CjjHttpRequest
        at cn.shuhe.projectfoundation.CommonRequest$AjcClosure15.run(CommonRequest.java:1)
        at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:149)
        at com.dataseed.cjjanalytics.aspect.MethodTimeTracker.requestTracker(MethodTimeTracker.java:29)
        at cn.shuhe.projectfoundation.CommonRequest.checkDeviceInfo(CommonRequest.java:223)
        at com.dataseed.huanbei.ui.BootActivity.startUp(BootActivity.java:296)
        at com.dataseed.huanbei.ui.BootActivity.onCreate_aroundBody2(BootActivity.java:129)
        at com.dataseed.huanbei.ui.BootActivity$AjcClosure3.run(BootActivity.java:1)
        at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:149)
        at com.dataseed.cjjanalytics.aspect.CjjTraceAspect.weaveActivityOnCreateJoinPoint(CjjTraceAspect.java:225)
        at com.dataseed.huanbei.ui.BootActivity.onCreate(BootActivity.java:126)
        at android.app.Activity.performCreate(Activity.java:7136)
        at android.app.Activity.performCreate(Activity.java:7127)


类转换异常是怎么肥事？
        

```

