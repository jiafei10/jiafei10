
package com.oki.ofits.common.api.extif;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.locks.ReentrantLock;
import java.util.function.Predicate;

import com.oki.ofits.common.logging.AppLogger;
import com.oki.ofits.common.message.MsgId;

/**
 * キューサービスクラス.<br>
 * 複数のキューをキーと紐づけて管理する。それぞれのキューは並列に操作が可能。
 */
public class QueueService<T extends QueueNode> {

    // キーとそれに対応するキュー（キューのラッパー）を保持するマップ
    private ConcurrentHashMap<String, QueueWrapper<T>> queueMap = new ConcurrentHashMap<>();

    // 全てのキューに格納されている要素の合計数
    private final AtomicLong totalSize = new AtomicLong();

    // キューに要素が追加されるまで待機する時間（ミリ秒単位）
    private final long pollWaitTime;

    /** アプリケーションロガー. */
    private AppLogger logger = AppLogger.getLogger(QueueService.class);

    public QueueService(long pollWaitTime) {
        this.pollWaitTime = pollWaitTime;
    }

    /**
     * 指定したキーのキューへ要素を追加する。
     * 
     * @param queueKey 要素を追加するキューのキー
     * @param node     追加する要素
     * @return 要素を処理するスレッドがあるかどうか
     */
    public boolean add(String queueKey, T node) {

        // 【初期化処理】
        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = new LinkedHashMap<>();

        // キーに対応するキューのラッパーを取得・生成
        QueueWrapper<T> queueWrapper = getOrCreateQueueWrapper(queueKey);

        queueWrapper.lockObj.lock();
        try {
            // 要素を処理する要求スレッド(pollメソッドを呼ぶスレッド)を特定する
            // [前提条件]：要素がキューに格納される前に、その要素を待つ要求スレッドが未受信スレッド待ち行列に並んでいること。
            boolean existsProcessor = false;

            // 未受信スレッド待ち行列を先頭から順に走査
            for (int i = 0; i < queueWrapper.unreceivedThreadQueue.size(); i++) {
                ThreadWrapper<T> threadWrapper = queueWrapper.unreceivedThreadQueue.get(i);

                // 当該スレッドで処理する要素か
                if (threadWrapper.filterFunc.test((node))) {
                    // 当該スレッドで処理する要素である

                    // 要素へ当該スレッドのリクエストIDをマーク
                    // （このマークを頼りに要求スレッドは、自スレッドで処理する要素をキューから取り出す）
                    node.processorReqId = threadWrapper.reqId;
                    existsProcessor = true;

                    // 当該スレッドを待ち行列から除く
                    queueWrapper.unreceivedThreadQueue.remove(i);

                    // 処理するスレッドが決まったため、ループを抜ける
                    break;

                } else {
                    // 当該スレッドで処理する要素でない
                    // 何もしない。
                }
            }

            // 要素を処理するスレッドが存在したか
            if (existsProcessor) {
                // 要素を処理するスレッドが存在した

                // 要素キューに要素を追加
                queueWrapper.nodeQueue.add(node);
                totalSize.incrementAndGet();

                debugInfo.clear();
                debugInfo.put("totalSize", totalSize.get());
                debugInfo.put("node", node);
                logger.debug(callerClazz, String.format("[受信スレッド] キュー(%s)に要素を追加しました。", queueKey), debugInfo);

            } else {
                // 要素を処理するスレッドが存在しなかった

                debugInfo.clear();
                debugInfo.put("node", node);
                logger.debug(callerClazz,
                        String.format("[受信スレッド] この要素を処理するスレッドが存在しないため、キュー(%s)に追加せず通知電文として扱います。", queueKey),
                        debugInfo);

                return false;
            }

        } finally {
            queueWrapper.lockObj.unlock();
        }
    
        return true;
    }

    /**
     * 指定したキーのキューから`filterFunc`の条件を満たす要素を取り出す。
     * もし、該当の要素がキューに存在しない場合は、該当の要素がキューに追加されるか、規定時間が経過するまで待機する。
     *
     * @param queueKey 要素を取得するキューのキー
     * @param filterFunc 現在のスレッド（リクエスト）が処理する要素か判定する関数
     * @return キューから取得した要素。タイムアウト時はnullを返す。
     */
    public T poll(String queueKey, Predicate<T> filterFunc) {

        // 【初期化処理】
        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = new LinkedHashMap<>();

        // キーに対応するキューのラッパーを取得・生成
        QueueWrapper<T> queueWrapper = getOrCreateQueueWrapper(queueKey);

        // リクエストを一意に識別するためのIDを生成
        // [スレッドID + "-" + UUID (一意であれば良いため、性能考慮でUUID v4)]
        String currentReqId = Thread.currentThread().getId() + "-" + UUID.randomUUID().toString();

        //  現在のスレッドが待ち行列に追加済みか
        boolean enqueued = false;
        // 待機開始時間
        long waitStartTime = System.currentTimeMillis();
        // スリープ時間
        long sleepTime = 50;
        // キューから取り出した要素
        T extractedNode = null;

        // 指定した時間、目的の要素が追加されるまで待機する（無限ループ）
        logger.debug(callerClazz,
                String.format("[待機スレッド(%s)] キュー(%s)にこのスレッドが処理する要素が格納されるまで待機します。", currentReqId, queueKey), null);

        WAIT_LOOP: do {
            queueWrapper.lockObj.lock();
            try {
                // 現在のスレッドが待ち行列に追加済みか
                // ※キューへの追加と待機のロックを分けると、スレッドの追い抜きが発生する可能性があるため、
                //   キューへの追加を無限ループの中で行う。
                if (!enqueued) {
                    // 現在のスレッドを未受信スレッド待ち行列に並ばせる
                    ThreadWrapper<T> threadWrapper = new ThreadWrapper<>();
                    threadWrapper.reqId = currentReqId;
                    threadWrapper.filterFunc = filterFunc;
                    queueWrapper.unreceivedThreadQueue.add(threadWrapper);
                    enqueued = true;
                }

                // 要素キューを先頭から順に走査
                for (int i = 0; i < queueWrapper.nodeQueue.size(); i++) {
                    T node = queueWrapper.nodeQueue.get(i);

                    // 現在のスレッドが処理する要素か
                    if (currentReqId.equals(node.processorReqId)) {
                        // 現在のスレッドが処理する要素である

                        // キューから要素を取り出す
                        extractedNode = node;
                        queueWrapper.nodeQueue.remove(i);
                        totalSize.decrementAndGet();

                        debugInfo.clear();
                        debugInfo.put("totalSize", totalSize.get());
                        debugInfo.put("node", node);
                        logger.debug(callerClazz,
                                String.format("[待機スレッド(%s)] キュー(%s)から要素を取り出しました。", currentReqId, queueKey), debugInfo);

                        // 待機解除するため、無限ループを抜ける。
                        break WAIT_LOOP;

                    } else {
                        // 現在のスレッドが処理する要素でない
                        // 何もしない。
                    }
                }

                // タイムアウト判定
                // （待機開始からの経過時間(現在時間 - 待機開始時間)が規定時間を超えたか。）
                if ((System.currentTimeMillis() - waitStartTime) > pollWaitTime) {
                    // タイムアウトした

                    // 現在のスレッドを未受信スレッド待ち行列から除く
                    for (int i = 0; i < queueWrapper.unreceivedThreadQueue.size(); i++) {
                        ThreadWrapper<T> threadWrapper = queueWrapper.unreceivedThreadQueue.get(i);

                        // 現在のスレッド（リクエスト）か
                        if (threadWrapper.reqId.equals(currentReqId)) {
                            // 現在のスレッド（リクエスト）である

                            // 未受信スレッド待ち行列から除く
                            queueWrapper.unreceivedThreadQueue.remove(i);
                            break;

                        } else {
                            // 現在のスレッド（リクエスト）でない
                            // 何もしない。
                        }
                    }

                    // 待機解除するため、無限ループを抜ける。
                    break WAIT_LOOP;

                } else {
                    // タイムアウトしていない
                    // 何もしない。
                }

                Thread.sleep(sleepTime);

            } catch (InterruptedException e) {
                // 【ログ出力】
                // エラーログを出力
                debugInfo.clear();
                debugInfo.put("reqId", currentReqId);
                debugInfo.put("sleepTime", sleepTime);
                logger.error(callerClazz, MsgId.EAE_03_00060, Arrays.asList("スリープ"), debugInfo);

                // 再割込み処理
                Thread.currentThread().interrupt();

            } finally {
                queueWrapper.lockObj.unlock();
            }

        } while (true);

        // タイムアウト判定
        if (extractedNode == null) {
            // タイムアウトした
            logger.debug(callerClazz, String.format("[待機スレッド(%s)] 規定時間内にキュー(%s)にこのスレッドが待つ要素が格納されなかったためタイムアウトしました。",
                    currentReqId, queueKey), null);

        } else {
            // タイムアウトしていない
            // 何もしない。
        }

        return extractedNode;
    }

    /**
     * 指定されたキーに対応するキュー（キューのラッパー）を取得する。存在しない場合は新規にキューを作成する。
     * 
     * @param queueKey キューのキー
     * @return キーに対応するキュー（キューのラッパー）
     */
    private QueueWrapper<T> getOrCreateQueueWrapper(String queueKey) {
        return queueMap.computeIfAbsent(queueKey, k -> {
            QueueWrapper<T> wrapper = new QueueWrapper<>();

            // FIFOで処理するため、リストへの追加順に並ぶArrayListを使用
            wrapper.nodeQueue = new ArrayList<>();
            wrapper.unreceivedThreadQueue = new ArrayList<>();

            // キューにアクセスした順に処理するため、公平ロックを有効にする(trueをセット)
            wrapper.lockObj = new ReentrantLock(true);

            return wrapper;
        });
    }

    private static class QueueWrapper<T> {

        /** 要素キュー. */
        public List<T> nodeQueue;
        /** 未受信スレッド待ち行列キュー. */
        public List<ThreadWrapper<T>> unreceivedThreadQueue;
        /** キューへのアクセスに排他制御をかけるためのロックオブジェクト. */
        public ReentrantLock lockObj;
    }

    private static class ThreadWrapper<T> {

        /** スレッドに付与されたリクエスト毎の一意なID（WEBサーバではスレッドが使いまわされることから、スレッドIDではなく、リクエスト単位に識別可能なIDを付与）. */
        public String reqId;
        /** スレッドが処理する要素か判定する関数. */
        public Predicate<T> filterFunc;
    }
}
