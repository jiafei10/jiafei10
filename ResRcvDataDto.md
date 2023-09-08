
package com.oki.ofits.common.api.extif.lsc;

import com.oki.ofits.common.api.extif.QueueNode;
import com.oki.ofits.logic.model.extif.MessageResultDto;
import com.oki.ofits.logic.model.extif.lsc.LscComHeaderDto;

/**
 * LSC応答受信データDTOクラス.
 * <p>
 * 要求電文に対する応答電文の情報をキューに格納するためのDTO
 * </p>
 */
public class LscResRcvDataDto extends QueueNode {

    /** 応答電文. */
    public Object rcvData;

    /** 応答電文のLSC共通ヘッダDTO. */
    public LscComHeaderDto rcvDataLscComHeaderDto;

    /** 応答電文のCHコード. */
    public String chCode;

    /** 電文実行結果DTO. */
    public MessageResultDto messageResultDto;

}
