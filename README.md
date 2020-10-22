# DownloadRange

  public class TaaHttpMgr {

      public static final TaaHttpClient CLIENT = new TaaHttpClient.Builder()
              .connectTimeout(10, TimeUnit.SECONDS)
              .readTimeout(10, TimeUnit.SECONDS)
              .writeTimeout(10, TimeUnit.SECONDS)
              .build();
      private static volatile SoftReference<Call> softReference = null;

      public static void downloadFileRange(@NonNull String url, final long startPoint,
                                           @NonNull final String fileDir,
                                           @NonNull final String fileName,
                                           @NonNull final DownloadRangeListener listener) {
          if (TextUtils.isEmpty(url) || startPoint < 0 || TextUtils.isEmpty(fileDir)
                  || TextUtils.isEmpty(fileName)) {
              listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_STATUS_ERR,
                      "downloadFileRange invalid params");
              return;
          }

          Request.Builder builder = new Request.Builder();
          Request rangeRequest = builder.url(url).addHeader("RANGE",
                  "bytes=" + startPoint + "-").build();

          Call call = CLIENT.newCall(rangeRequest, "");
          softReference = new SoftReference<>(call);
          call.enqueue(new Callback() {
              @Override
              public void onFailure(Call call, IOException e) {
                  TaesLog.d(TAG, "downloadFileRange.onFailure: " + e.getMessage());
                  if (e instanceof UnknownHostException || e instanceof ConnectException
                          || e instanceof SocketTimeoutException) {
                      listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_NET_WORK_ERR, e.getMessage());
                  } else {
                      listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_STATUS_ERR, e.getMessage());
                  }
              }

              @Override
              public void onResponse(Call call, Response response) {
                  File file = new File(fileDir + "/" + fileName);
                  byte[] buff = new byte[4096];
                  ResponseBody body = response.body();
                  if (body == null) {
                      listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_STATUS_ERR,
                              "response.body null");
                      return;
                  }
                  try (InputStream inputStream = body.byteStream()) {
                      RandomAccessFile accessFile = new RandomAccessFile(
                              fileDir + "/" + fileName, "rwd");
                      long total = body.contentLength();
                      accessFile.seek(startPoint);
                      long downloadSum = startPoint;

                      for (; ; ) {
                          int readLength = inputStream.read(buff);
                          if (readLength == -1) {
                              break;
                          }
                          accessFile.write(buff, 0, readLength);
                          downloadSum += readLength;

                          if (file.exists()) {
                              listener.onDownloading(fileName, downloadSum);
                          } else {
                              TaesLog.d(TAG, "onResponse: reading Stop Download File not exists");
                              listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_FILE_NOT_EXIST,
                                      "File not Exists");
                              return;
                          }
                      }
                      if (downloadSum - startPoint == total) {
                          TaesLog.d(TAG, "onResponse: all length check");
                          listener.onDownloadSuccess(fileName);
                      } else {
                          TaesLog.w(TAG, "onResponse: all length check Err!");
                          listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_STATUS_ERR,
                                  "download all length err!");
                      }
                  } catch (Exception e) {
                      TaesLog.e(TAG, "TaaHttpMgr.onResponse:Exception: ", e);
                      listener.onDownloadFailed(fileName, DOWNLOAD_FAILED_STATUS_ERR, e.getMessage());
                  }
              }
          });
      }

      public static void stopDownloadRange() {
          if (softReference != null) {
              Call call = softReference.get();
              if (call != null) {
                  if (!call.isCanceled()) {
                      call.cancel();
                  }
              }
          }
      }

      public interface DownloadRangeListener {
          void onDownloadSuccess(String fileName);

          void onDownloading(String fileName, long downloadSum);

          void onDownloadFailed(String fileName, @SotaDownloadErrStatus int errCode, String errMsg);
      }
  }
