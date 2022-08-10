


用 Android 的 AsyncTask 来解决线程间通信：
```java
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
    protected Long doInBackground(URL... urls) {
        int count = urls.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(urls[i]);
            publishProgress((int) ((i / (float) count) * 100));
            // Escape early if cancel() is called
            if (isCancelled()) break;
        }
        return totalSize;
    }

    protected void onProgressUpdate(Integer... progress) {
        setProgressPercent(progress[0]);
    }

    protected void onPostExecute(Long result) {
        showDialog("Downloaded " + result + " bytes");
    }
}
```
AsyncTask 是 Android 对线程池 Executor 的封装，但它的缺点也很明显：

- 需要处理很多回调，如果业务多则容易陷入「回调地狱」。
- 硬是把业务拆分成了前台、中间更新、后台三个函数。
