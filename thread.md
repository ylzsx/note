```java
public interface IThreadToUser {
    // 线程将分配的内存块给用户
    void getMemoryBlocks(int position, ArrayList<Integer> memoryBlocks);

    // LRU替换后通知界面更新
    void refreshInterface(int position, OperateType operateType);

    // 新建文件进行新建刷新
    void insertRefresh(FileResponse response);

    // 删除完后进行通知界面刷新
    void deleteRefresh(int position);
}
```

```java
// DataDeleteThread.java
public class DataDeleteThread extends CustomThread {

    private IThreadToUser mIThreadToUser;

    public void setIThreadToUser(IThreadToUser IThreadToUser) {
        mIThreadToUser = IThreadToUser;
    }

    /**
     * 删除文件
     * @param ufdId
     * @param position
     */
    public void deleteData(int ufdId, int position) {
        DataDeleteThread thread = this;
        // 1. 查看该文件是否被打开
        Map<Integer, CustomThread> threadRunningQueue = ThreadManager.getInstance().getThreadRunningQueue();
        if (threadRunningQueue.get(position) != null) {    // 说明该文件被打开
            ToastUtil.showToast("文件被打开，删除请先关闭文件");
        } else {
            // 2. 查询阻塞队列中是否有该文件，有则移除
            Map<Integer, CustomThread> threadBlockedQueue = ThreadManager.getInstance().getThreadBlockedQueue();
            for (Integer key : threadBlockedQueue.keySet()) {
                if (position == key) {
                    threadBlockedQueue.remove(key);
                    break;
                }
            }

            // 3. 执行删除操作
            Repository.getInstance().deleteFile(ufdId, new IOSListener.IDeleteFile() {
                @Override
                public void onDeleteFile(int response) {
                    mIThreadToUser.deleteRefresh(position);
                    // 删除完毕，停止关闭线程
                    thread.stop();
                }

                @Override
                public void onError(Throwable t) {
                    ToastUtil.showToast("删除失败");
                }
            });
        }
    }
}
```

```java
public class DataGenerationThread extends CustomThread {

    private IThreadToUser mIThreadToUser;

    public void setIThreadToUser(IThreadToUser IThreadToUser) {
        mIThreadToUser = IThreadToUser;
    }

    /**
     * 文件创建
     * 1. 创建目录项
     * 2. 创建文件
     * @return
     */
    public void generateData(FileDirRequest fileDirRequest) {
        DataGenerationThread thread = this;
        // 创建目录项
        Repository.getInstance().createFileDir(fileDirRequest, new IOSListener.ICreateFileDir() {
            @Override
            public void onCreateFileDir(int ufdId) {
                fileDirRequest.setUfdId(ufdId);
                // 创建文件
                Repository.getInstance().createFile(fileDirRequest, new ICreateFile() {
                    @Override
                    public void onCreateFile(int response) {
                        // TODO: 为防止内存信息丢失，不可全部刷新界面。之后可做优化
                        FileResponse file = new FileResponse();
                        file.setUfdId(ufdId);
                        file.setFileName(fileDirRequest.getFileName());
                        file.setUserId(fileDirRequest.getUserId());
                        file.setFileLength(fileDirRequest.getFileLength());
                        file.setIsOpenFlag(0);

                        mIThreadToUser.insertRefresh(file);
                        // 执行结束，数据生成线程停止
                        thread.stop();
                    }

                    @Override
                    public void onError(Throwable t) {}
                });
            }

            @Override
            public void onError(Throwable t) {}
        });
    }
}
```

```java
public class ExecuteThread extends CustomThread {

    // 为展示LRU置换过程，设置的一个缓存区
    private ArrayList<String> buffer = new ArrayList<>();
    private int page = 0;

    private IThreadToUser mThreadToUser;

    public void setThreadToUser(IThreadToUser threadToUser) {
        mThreadToUser = threadToUser;
    }

    public void readDisk(int ufdId, int position) {
        final ExecuteThread thread = this;
        ArrayList<Integer> memoryBlocks = this.memoryBlocks;
        // 1. 查看缓存区是否有
        if (!buffer.isEmpty()) {
            MemoryManager.getInstance().LRU(memoryBlocks, buffer.get(0));
            buffer.remove(0);
            // 通知界面更新
            mThreadToUser.refreshInterface(position, OperateType.OPEN);
        } else {
            // 2. 缓存区无，则发送网络请求，加入缓存区
            Repository.getInstance().searchFile(ufdId, page, new IOSListener.ISearchFileListener() {
                @Override
                public void onSearchFileListener(ArrayList<FileContentResponse> fileContentResponses) {
                    for (FileContentResponse fileContentRespons : fileContentResponses) {
                        buffer.add(fileContentRespons.getContent());
                    }
                    page++;
                    MemoryManager.getInstance().LRU(memoryBlocks, buffer.get(0));
                    buffer.remove(0);
                    thread.stop();
                    mThreadToUser.refreshInterface(position, OperateType.OPEN);
                }

                @Override
                public void onError(Throwable t) {}
            });
        }
    }

    public void operateFile(int ufdId, int position, OperateType operateType) {
        if (operateType == OperateType.OPEN) {
            openFile(ufdId, position);
        } else {
            closeFile(ufdId, position);
        }
    }

    /**
     * 关闭文件
     * 1. 释放内存
     * 2. 修改文件目录表
     * 3. 通知用户界面
     * 4. 结束当前进程
     * 5. 唤醒阻塞队列的线程
     * @param ufdId
     * @param position
     */
    private void closeFile(int ufdId, int position) {
        if(MemoryManager.getInstance().free(this.getMemoryBlocks()) > 0) {

            // 修改文件目录表
            Repository.getInstance().closeFile(ufdId, new IOSListener.ICloseFileListener() {
                @Override
                public void onCloseFileListener(int response) {
                    // 通知用户界面
                    mThreadToUser.refreshInterface(position, OperateType.CLOSE);
                    // 结束当前进程
                    ThreadManager.getInstance().getThreadRunningQueue().remove(position);

                    // 查询阻塞队列是否有线程，如果有的话便唤醒
                    Map<Integer, CustomThread> threadBlockedQueue = ThreadManager.getInstance().getThreadBlockedQueue();
                    for (Integer key : threadBlockedQueue.keySet()) {
                        ExecuteThread thread = (ExecuteThread) threadBlockedQueue.get(key);
                        threadBlockedQueue.remove(key);
                        thread.openFile(ufdId, key);
                        break;
                    }
                }

                @Override
                public void onError(Throwable t) {}
            });
        }
    }

    /**
     * 打开文件
     * 1. 查询内存是否够用
     * 2. 内存够用则分配内存，线程进入执行队列；否则线程进入阻塞队列
     * @param ufdId
     * @param position recyclerView item的位置
     */
    public void openFile(int ufdId, int position) {
        final ExecuteThread thread = this;
        final ArrayList<Integer> memoryBlocks = MemoryManager.getInstance().malloc();
        if (memoryBlocks != null && memoryBlocks.size() != 0) {     // 内存够用
            Repository.getInstance().openFile(ufdId, new IOSListener.IOpenFileListener() {
                @Override
                public void onOpenFileListener(int response) {
                    // 记录线程占用的内存块
                    thread.setMemoryBlocks(memoryBlocks);
                    // 线程进入执行队列
                    ThreadManager.getInstance().addThreadRunningQueue(position, thread);
                    // 返回当前线程占用的内存块号
                    mThreadToUser.getMemoryBlocks(position, memoryBlocks);
                }
                @Override
                public void onError(Throwable t) {
                    // 文件打开失败，释放内存
                    MemoryManager.getInstance().free(memoryBlocks);
                }
            });
        } else {    // 内存不够用，线程进入阻塞队列
            ThreadManager.getInstance().addThreadBlockedQueue(position, thread);
            ToastUtil.showToast("内存占用过大，请释放内存");
        }
    }
}
```



