# android常见bug及解决方案总结
### 空指针
##### 解决方案
* 不确定对象在使用前先做是否为空判断
* 特别注意：fragment getActivity为null处理

### 数组越界
##### 解决方案
* 使用索引值获取对象值时，需判断索引值是否小于数据源大小
example：
```javascript
if (mData != null && mData.size() !=0 && i < mData.size()){
  Obj obj = mData.get(i);
}
```

### ListView或RecycleView 更新数据源命令不同步
##### 解决方案
* 保证设置数据源和执行adapter.notifyDataSetChanged在同一个线程并且在Ui线程
example:
```javascript
adapter.setData(mData);
adapter.notifyDataSetChanged();
```

### Windows无页面附加（Unable to add window.....is your activity running?）
##### 解决方案
* 执行windows窗体Dialog或PopupWindow时，先判断当前页面是否销毁，若页面还在则可以执行窗体显示操作，可以通过全局变量或activity自定义堆栈管理判断当前页面是否销毁，在onDestory里面做关闭窗体操作并置空
example：
```javascript
@Override
protected void onDestroy() {    
if (mDialog != null && mDialog.isShowing()){        
mDialog.dismiss();        
mDialog = null;    
}    
super.onDestroy();
}
```

### sqlite问题（数据库缺字段或执行过程异常）
##### 解决方案
升级sqlite方法里面添加字段处理，此时记得加入try catch处理方式，防止出现崩溃现象。
example:
```javascript
FinalDb.DaoConfig daoConfig = new FinalDb.DaoConfig();
daoConfig.setContext(this);
daoConfig.setDbName(ChatDao.DATABASE_NAME);
daoConfig.setDebug(true);
daoConfig.setDbVersion(ChatDao.DATABASE_VERSION_CODE);
daoConfig.setDbUpdateListener(new FinalDb.DbUpdateListener() {
			@Override
			public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
				// 当之前版本的数据升级到新版版本的数据,我们需要给对应表增加新的字段
				try {
					// 添加字段has_appraised字段
					db.execSQL(ChatDao.VERSION_3_SQL_ADD_COLUMN_HAS_PRAISED);
				}
				catch (Exception e){
					e.printStackTrace();
				}

				try {
					// 添加字段receiveDate字段
					db.execSQL(ChatDao.VERSION_4_SQL_ADD_CLUMN_RECEIVE_DATE);
				}
				catch (Exception e){
					e.printStackTrace();
				}
			}
		});
FinalDb.create(daoConfig);
```
### OOM（内存溢出）
#### 情况一
处理bitmap资源不得当造成（压缩或者变换得到新bitmap）
##### 解决方案
采用try catch处理方式，若出现异常再做压缩，期间采用弱引用方式处理。
example：
    	
```javascript
        WeakReference<Bitmap> bitmapWeakReference;
    	// First decode with inJustDecodeBounds=true to check dimensions
		final BitmapFactory.Options options = new BitmapFactory.Options();
		try {
			options.inJustDecodeBounds = true;
			BitmapFactory.decodeFileDescriptor(fd, null, options);

			// Calculate inSampleSize
			options.inSampleSize = calculateInSampleSize(options, reqWidth,
					reqHeight);

			// Decode bitmap with inSampleSize set
			options.inJustDecodeBounds = false;
			// 弱引用处理图片 add leibing 2016/12/8
			bitmapWeakReference
					= new WeakReference<Bitmap>(BitmapFactory.decodeFileDescriptor(fd, null, options));
			if (bitmapWeakReference != null && bitmapWeakReference.get() != null)
				return bitmapWeakReference.get();
		}catch (OutOfMemoryError ex){
			// 压缩图片指定值 add by leibing 2016/12/8
			options.inSampleSize = options.inSampleSize + 4;
			options.inJustDecodeBounds = false;
			// 弱引用处理图片 add leibing 2016/12/8
			bitmapWeakReference
					= new WeakReference<Bitmap>(BitmapFactory.decodeFileDescriptor(fd, null, options));
			if (bitmapWeakReference != null && bitmapWeakReference.get() != null)
				return bitmapWeakReference.get();
		}
```
#### 情况二
各种内存泄漏问题造成，主要有以下：
##### 单例造成的内存泄露
###### 解决方案
1、将该属性的引用方式改为弱引用;
2、如果传入Context，使用ApplicationContext;
example
泄漏代码片段   	
```javascript
    // sington
    private static InstanceClass instance;
    // activity refer
    private Context mContext;
    
    /**
     * set activity refer
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param mContext activity refer
     * @return
     */
    public void setRefer(Context mContext){
        this.mContext = mContext;
    }
    
    /**
     * constructor
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param
     * @return
     */
    private InstanceClass(){
    }

    /**
     * sington
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param
     * @return
     */
    public static InstanceClass getInstance(){
        if (instance == null){
            synchronized (InstanceClass.class){
                if (instance == null)
                    instance = new InstanceClass();
            }
        }

        return instance;
    }
```
Solution：使用WeakReference
    	
```javascript
    // sington
    private static InstanceClass instance;
    // activity refer
    private WeakReference<Context> mContextWeakRef;

    /**
     * set activity refer
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param mContext activity refer
     * @return
     */
    public void setRefer(Context mContext){
        mContextWeakRef = new WeakReference<Context>(mContext);
    }
    
    /**
     * constructor
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param
     * @return
     */
    private InstanceClass(){
    }

    /**
     * sington
     * @author leibing
     * @createTime 2016/12/9
     * @lastModify 2016/12/9
     * @param
     * @return
     */
    public static InstanceClass getInstance(){
        if (instance == null){
            synchronized (InstanceClass.class){
                if (instance == null)
                    instance = new InstanceClass();
            }
        }

        return instance;
    }
```
##### InnerClass匿名内部类
###### 解决方案
1、将内部类变成静态内部类;
2、如果有强引用Activity中的属性，则将该属性的引用方式改为弱引用;
3、在业务允许的情况下，当Activity执行onDestory时，结束这些耗时任务;
example
泄漏代码片段
    	
```javascript
public class InnerClassActivity extends Activity{

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // start a thread to work
        new InnerThread().start();
    }
    
    /**
     * @interfaceName: InnerThread
     * @interfaceDescription: custom thread
     * @author: leibing
     * @createTime: 2016/12/9
     */
    class InnerThread extends Thread{
        @Override
        public synchronized void start() {
            super.start();
        }

    }
}
```
Solution：使用WeakReference + static
    	
```javascript
public class InnerClassActivity extends Activity{
    // 图片资源
    private Drawable mDrawable;
    // inner thread
    private InnerThread mInnerThread;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // init drawable
        mDrawable = getResources().getDrawable(R.drawable.ic_launcher);
        // start a thread to work
        mInnerThread = new InnerThread(mDrawable);
        mInnerThread.start();
    }
    
    @Override
    protected void onDestroy() {
        if (mInnerThread != null)
            mInnerThread.setIsRun(false);
        super.onDestroy();
    }
    
    /**
     * @interfaceName: InnerThread
     * @interfaceDescription: custom thread
     * @author: leibing
     * @createTime: 2016/12/9
     */
    static class InnerThread extends Thread{
        // weak ref
        public WeakReference<Drawable> mDrawableWeakRef;
        // is run
        private boolean isRun = true;

        /**
         * constructor
         * @author leibing
         * @createTime 2016/12/9
         * @lastModify 2016/12/9
         * @param mDrawable 图片资源（存在对页面的引用风险）
         * @return
         */
        public InnerThread(Drawable mDrawable){
          mDrawableWeakRef = new WeakReference<Drawable>(mDrawable);
        }
        
        /**
         *  set is run
         * @author leibing
         * @createTime 2016/12/9
         * @lastModify 2016/12/9
         * @param isRun
         * @return
         */
        public void setIsRun(boolean isRun){
            this.isRun = isRun;
        }

        @Override
        public void run() {
            while (isRun){
                // do work
                try {
                    // sleep one second
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        @Override
        public synchronized void start() {
            super.start();
        }
    }
}
```
#####Activity Context 的不正确使用
###### 解决方案
1、使用ApplicationContext代替ActivityContext，因为ApplicationContext会随着应用程序的存在而存在，而不依赖于activity的生命周期；
2、对Context的引用不要超过它本身的生命周期，慎重的对Context使用“static”关键字。Context里如果有线程，一定要在onDestroy()里及时停掉。
example
泄漏代码片段   	
```javascript
public class DrawableActivity extends Activity{
    // static drawable 
    private static Drawable leakDrawable;
    
    @Override
    protected void onCreate(Bundle state) {
        super.onCreate(state);
        TextView label = new TextView(this);
        // init drawable
        if (leakDrawable == null) {
            leakDrawable = getResources().getDrawable(R.drawable.ic_launcher);
        }
        // view set drawable
        label.setBackgroundDrawable(leakDrawable);
        
        setContentView(label);
    }
}
```
Solution：    	
```javascript
public class DrawableActivity extends Activity{
    // static drawable
    private static Drawable leakDrawable;

    @Override
    protected void onCreate(Bundle state) {
        super.onCreate(state);
        TextView label = new TextView(this);
        // init drawable
        if (leakDrawable == null) {
            leakDrawable = getApplicationContext().getResources().getDrawable(R.drawable.ic_launcher);
        }
        // view set drawable
        label.setBackgroundDrawable(leakDrawable);

        setContentView(label);
    }
}
```
##### Handler引起的内存泄漏
###### 解决方案
1、可以把Handler类放在单独的类文件中，或者使用静态内部类便可以避免泄露;
2、如果想在Handler内部去调用所在的Activity,那么可以在handler内部使用弱引用的方式去指向所在Activity.使用Static + WeakReference的方式来达到断开Handler与Activity之间存在引用关系的目的。
example
泄漏代码片段 
    	
```javascript
public class HandlerActivity extends Activity{
    // custom handler
    private CustomHandler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // init custom handler
        mHandler = new CustomHandler();
        // sendMsg
        Message msg = new Message();
        mHandler.sendMessage(msg);
    }
    
    /**
     * @interfaceName: CustomHandler
     * @interfaceDescription: custom handler
     * @author: leibing
     * @createTime: 2016/12/9
     */
    class CustomHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    }
}
```
Solution（static + weakRef）：    
    	
```javascript
public class HandlerActivity extends Activity{
    // custom handler
    private CustomHandler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // init custom handler
        mHandler = new CustomHandler(this);
        // sendMsg
        Message msg = new Message();
        mHandler.sendMessage(msg);
    }

    /**
     * @interfaceName: CustomHandler
     * @interfaceDescription: custom handler
     * @author: leibing
     * @createTime: 2016/12/9
     */
    static class CustomHandler extends Handler{
        // weak ref
        private WeakReference<Context> mContextWeakRef;
        
        /**
         * constructor
         * @author leibing
         * @createTime 2016/12/9
         * @lastModify 2016/12/9
         * @param mContext activity ref
         * @return
         */
        public CustomHandler(Context mContext){
            mContextWeakRef = new WeakReference<Context>(mContext);    
        }
        
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (mContextWeakRef != null && mContextWeakRef.get() != null){
                // do work
            }
        }
    }
}
```
##### 注册监听器的泄漏
###### 解决方案
1、使用ApplicationContext代替ActivityContext;
2、在Activity执行onDestory时，调用反注册;

##### Cursor，Stream没有close，View没有recyle；
###### 解决方案
资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于 java虚拟机内，还存在于java虚拟机外。如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄漏。因为有些资源性对象，比如SQLiteCursor(在析构函数finalize(),如果我们没有关闭它，它自己会调close()关闭)，如果我们没有关闭它，系统在回收它时也会关闭它，但是这样的效率太低了。因此对于资源性对象在不使用的时候，应该调用它的close()函数，将其关闭掉，然后才置为null. 在我们的程序退出时一定要确保我们的资源性对象已经关闭。

##### 集合中对象没清理造成的内存泄漏
###### 解决方案
在Activity退出之前，将集合里的东西clear，然后置为null，再退出程序。
    	
```javascript
private List<String> mData;    
public void onDestory() {        
    if (mData!= null) {
        mData.clear();
        mData= null;
    }
}
```
##### WebView造成的泄露
###### 解决方案
为webView开启另外一个进程，通过AIDL与主线程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，从而达到内存的完整释放。

### 类型转换错误
#### 解决方案
首先判断类型转换前数据是否符合转化后的数据再做处理。
example：
字符串转整型数字
首先判断字符串是否为数字字符串，然后再转换，此时最好加上try catch处理，防止崩溃。

### Android Studio: GC overhead limit exceeded
#### 报错现象
```javascript
Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
> java.lang.OutOfMemoryError: GC overhead limit exceeded
```
#### 解决方案
往module gradle里面添加代码

dexOptions {
    javaMaxHeapSize "4g"
}

code:

```javascript
android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "com.xxxxx.android"
        minSdkVersion 16
        targetSdkVersion 23
        versionCode 8
        versionName "1.3"
    }

    dexOptions {
        javaMaxHeapSize "4g"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
链接：http://stackoverflow.com/questions/35034830/android-studio-gc-overhead-limit-exceeded


以上是笔者在项目中遇到的常见bug以及解决方案，如有不足，请补充。

### 关于作者
* QQ:872721111
* Email:leibing1989@126.com
* Github:https://github.com/leibing8912