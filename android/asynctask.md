        android在新版本中不允许UI线程访问网络；在android中UI线程中不能执行耗时太长的任务，否则会引发ANR。

        不同线程间的通信，可以选择Handler或者AsyncTask解决线程间通信问题。这里主要介绍AsyncTask。

AsyncTask是一个抽象类含有三个参数， AsyncTask&lt;Params, Progress, Result&gt;，根据参数的名字可以知道分别为：输入参数，进度，与返回结果，如果你要使用它必须要集成他，或者采用匿名类的方式。

## Samples：

```
AsyncTask<Integer, Void, Integer> task = new AsyncTask<Integer, Void, Integer>() {

			@Override
			protected Integer doInBackground(Integer... params) {
				return params[0] * params[0];
			}

			@Override
			protected void onProgressUpdate(Void... values) {
				super.onProgressUpdate(values);
			}

			@Override
			protected void onCancelled() {
				super.onCancelled();
			}
			
			@Override
			protected void onPreExecute() {
				super.onPreExecute();
			}

			@Override
			protected void onPostExecute(Integer result) {
				Toast.makeText(AsyncTaskActivity.this, "Current Value = " + result, Toast.LENGTH_LONG).show();
				setTextValue("Current Value =" + result);
			}
		};
		if (Build.VERSION.SDK_INT >= 11) {
			task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,value);
		} else {
			task.execute(value);
		}
```

1： protected Integer doInBackground\(Integer... params\)是必须要重写的，因为它是在后台执行的，耗时逻辑和网络访问都可以放到这个里面，其中返回类型就是AsyncTask中的第三个参数的类型，输入参数是可变参数类型，与AsyncTask第一个参数保持一致，如果没有参数可以采用Void代替，V大写。

2：void onPreExecute\(\)  这个表示在后台线程执行之前，UI可以执行的操作，一般用来开始进度条，或者进行提醒。

3：void onProgressUpdate\(Void... values\) ，根据名字也可以看出是调整进度条的，输入的参数与AsyncTask第二个参数一致，用来在UI界面更新进度条。

4： void onPostExecute\(Integer result\) ，该函数表示执行完毕后，返回的内容，已经在UI线程执行了，可以用来更新界面。

最后来看执行的部分加入了一个判断，这是因为android刚推出AsyncTask的时候，采用的单线程方式，前一个执行完了后再执行下一个，后来在2.x版本中改为了多个线程，到3.0版本时又改为了一个线程。为了防止阻塞，在3.0以后的版本中指定执行的Executor。



## AsyncTask源码分析（Android4.4.2）





