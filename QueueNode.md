
package com.oki.ofits.common.api.extif;

/**
 * キュー要素クラス.
 */
public class QueueNode {

    /**
     * この要素を処理するスレッドへ付与されたリクエストID<br>
     * ※QueueServiceクラスが自動設定する。
     */
    public String processorReqId;
}
