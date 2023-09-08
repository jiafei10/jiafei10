
package com.oki.ofits.common.api.extif.lsc;

import java.util.Objects;

import org.springframework.stereotype.Component;

import com.oki.ofits.common.api.extif.QueueService;
import com.oki.ofits.common.constant.Const;
import com.oki.ofits.common.constant.SystemCode;
import com.oki.ofits.logic.model.extif.MessageResultDto;
import com.oki.ofits.logic.model.extif.lsc.LscComHeaderDto;

/**
 * LAN信号変換装置要求応答レシーバークラス.
 */
@Component
public class LscReqResReceiver {

    /** 応答電文キュー */
    private final QueueService<LscResRcvDataDto> responseQueueService;

    public LscReqResReceiver() {
        // pollWaitTime：15秒(仮値) TODO:要設定値見直し
        responseQueueService = new QueueService<LscResRcvDataDto>(Const.RECEIVER_POLL_WAIT_TIME);
    }

    /**
     * 要求電文に対する応答電文を受信するまで待機する.
     * 
     * @param reqLscComHeaderDto 要求電文のLSC共通ヘッダDTO
     * @param chCode CHコード
     * @return LscResRcvDataDto LSC応答受信データDTO
     */
    public LscResRcvDataDto waitResponse(LscComHeaderDto reqLscComHeaderDto, String chCode) {

        // 要求電文に対応する応答電文の電文識別を取得
        String convertTelegramId = reqToResTelegramId(reqLscComHeaderDto.telegramIdent);
        
        // 監視するキュー（応答電文が格納されるキュー）のキーを生成
        String queueKey = createQueueKey(convertTelegramId, chCode);

        // 要求電文の情報から、要求電文のIDを生成
        String reqTelegramId = createTelegramId(convertTelegramId, chCode);

        // 監視するキューに応答電文が追加されるか、指定の時間が経過するまで待機
        LscResRcvDataDto response = responseQueueService.poll(queueKey, (node) -> {

            if (node.chCode == null) {
                // 応答電文にCHコードが存在されていないため、電文識別で要求と応答の紐づけを行う

                return node.rcvDataLscComHeaderDto.telegramIdent.equals(convertTelegramId);

            } else {
                // 応答電文にCHコードが存在されているため、電文識別とCHコードで要求と応答の紐づけを行う

                // 応答電文のIDを生成
                String resTelegramId = createTelegramId(node.rcvDataLscComHeaderDto.telegramIdent, node.chCode);

                // 対象の応答電文か判定
                return reqTelegramId.equals(resTelegramId);
            }
        });

        if (response == null) {
            // タイムアウトした場合

            // タイムアウト用の応答を生成
            response = new LscResRcvDataDto();
            response.rcvData = null;
            response.rcvDataLscComHeaderDto = null;
            response.chCode = null;
            response.messageResultDto = new MessageResultDto();
            response.messageResultDto.resultCd = SystemCode.ResultIfCode.CONNECT_ERROR; // 接続エラー
        }

        return response;
    }

    /**
     * 受信した応答電文をセットする.
     * 
     * @param response LSC応答受信データDTO
     * @return 応答電文セット結果
     */
    public boolean setResponse(LscResRcvDataDto response) {

        // 応答電文を格納するキューのキーを生成
        String queueKey = createQueueKey(response.rcvDataLscComHeaderDto.telegramIdent, response.chCode);

        // キューに応答電文を追加
        return responseQueueService.add(queueKey, response);
    }

    /**
     * 電文の情報から、応答電文を格納するキューのキーを生成する.
     * 
     * @param telegramIdent 電文識別
     * @param chCode CHコード
     * @return 電文を格納するキューのキー
     */
    private String createQueueKey(String telegramIdent, String chCode) {

        // 複数装置を並列に処理するため、装置毎にキューを設ける。

        String queueKey = "";

        if (Objects.isNull(chCode)) {

            // キューのキーを生成
            queueKey =
                    String.format("LscTelegramIdent(%s)", telegramIdent);
        } else {

            // キューのキーを生成
            queueKey =
                    String.format("LscTelegramIdent-ChCode(%s-%s)", telegramIdent, chCode);
        }

        return queueKey;
    }

    /**
     * 電文の情報から、電文IDを生成する.
     * 
     * @param telegramIdent 電文識別
     * @param chCode CHコード
     * @return 電文ID
     */
    private String createTelegramId(String telegramIdent, String chCode) {

        // 電文IDを生成
        String telegramId = "";

        // CHコードが存在しない場合、電文識別 で紐づける。
        if (Objects.isNull(chCode)) {

            telegramId = telegramIdent;

        } else {

            telegramId = String.format("%s-%s", telegramIdent, chCode);
        }

        return telegramId;
    }

    /**
     * 要求電文と応答電文の電文識別対応表.
     * 
     * @param telegramIdent 要求電文の電文識別
     * @return 応答電文の電文識別
     */
    private String reqToResTelegramId(String reqTelegramId) {

        // 電文IDを生成
        String resTelegramId = "";

        switch (reqTelegramId) {

        // 要求電文の電文識別が"550411(時刻通知) "の場合、応答電文の電文識別"550461(時刻要求) "を指定
        case Const.IF_LSC_TELE_ID_TIME_NOTICE:

            resTelegramId = Const.IF_LSC_TELE_ID_TIME_REQ;

            break;

        // LscGateway.sndBaseStaSelectStatusReq 基地局選択要求->基地局選択状態通知
        // 要求電文の電文識別が"550413(基地局選択要求) "の場合、応答電文の電文識別"550463(基地局選択状態通知) "を指定
        case Const.IF_LSC_TELE_ID_BASE_STA_SELECT_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_BASE_STA_SELECT_STATUS_NOTICE;

            break;

        // LscGateway.sndReglationCtrlStatusReq 規制制御要求->規制制御状態通知
        // 要求電文の電文識別が"550424(規制制御要求) "の場合、応答電文の電文識別"550474(規制制御状態通知) "を指定
        case Const.IF_LSC_TELE_ID_REGULATION_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_REGULATION_STATUS_NOTICE;

            break;

        // 要求電文の電文識別が"550421(運用開始応答通知) "の場合、応答電文の電文識別"550471(運用開始要求) "を指定
        case Const.IF_LSC_TELE_ID_OPE_START_RES_NOTICE:

            resTelegramId = Const.IF_LSC_TELE_ID_OPE_START_REQ;

            break;

        // LscGateway.sndLscCommConfReq 通信設定要求->通信設定応答通知
        // 要求電文の電文識別が"550422(通信設定要求) "の場合、応答電文の電文識別"550472(通信設定応答通知) "を指定
        case Const.IF_LSC_TELE_ID_TELECOM_CONF_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_TELECOM_CONF_RES_NOTICE;

            break;

        // LscGateway.sndLscCommStartReq 通信開始要求->通信開始応答通知
        // 要求電文の電文識別が"550428(通信開始要求) "の場合、応答電文の電文識別"550478(通信開始応答通知) "を指定
        case Const.IF_LSC_TELE_ID_TELECOM_START_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_TELECOM_START_RES_NOTICE;

            break;

        // LscGateway.sndLscCommEndReq 通信終了要求->通信終了応答通知
        // 要求電文の電文識別が"550429(通信終了要求) "の場合、応答電文の電文識別"550479(通信終了応答通知) "を指定
        case Const.IF_LSC_TELE_ID_TELECOM_END_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_TELECOM_END_RES_NOTICE;

            break;

        // LscGateway.sndSelcallReq セレコール応答通知->セレコール応答受信通知
        // 要求電文の電文識別が"550419(セレコール応答通知) "の場合、応答電文の電文識別"550469(セレコール応答受信通知) "を指定
        case Const.IF_LSC_TELE_ID_SELECALL_RES_NOTICE:

            resTelegramId = Const.IF_LSC_TELE_ID_SELECALL_RES_RECV_NOTICE;

            break;

        // 要求電文の電文識別が"800001(時刻設定変更要求) "の場合、応答電文の電文識別"800101(時刻設定変更応答) "を指定
        case Const.IF_LSC_TELE_ID_TIME_CONF_CHANGE_REQ:

            resTelegramId = Const.IF_LSC_TELE_ID_TIME_CONF_CHANGE_RES;

            break;

        // 要求電文の電文識別が"800003(チャネル占有通信設定) "の場合、応答電文の電文識別"800103(チャネル占有通信設定応答) "を指定
        case Const.IF_LSC_TELE_ID_CHANNEL_TELECOM_CONF:

            resTelegramId = Const.IF_LSC_TELE_ID_CHANNEL_TELECOM_CONF_RES;

            break;

        // 要求電文の電文識別が"800013(移動局在圏情報取得要求) "の場合、応答電文の電文識別"800113(移動局在圏情報取得応答) "を指定
        case Const.IF_LSC_TELE_ID_MOBILE_STA_AREA_RES:

            resTelegramId = Const.IF_LSC_TELE_ID_MOBILE_STA_AREA_REQ;

            break;

        // 要求電文の電文識別が"800014(バージョン要求) "の場合、応答電文の電文識別"800114(バージョン通知) "を指定
        case Const.IF_LSC_TELE_ID_VER_NOTICE:

            resTelegramId = Const.IF_LSC_TELE_ID_VER_REQ;

            break;

        // 要求電文の電文識別が"800015(LAN信号変換装置系切替要求) "の場合、応答電文の電文識別"800115(LAN信号変換装置系切替応答) "を指定
        case Const.IF_LSC_TELE_ID_LAN_SWITCH_RES:

            resTelegramId = Const.IF_LSC_TELE_ID_LAN_SWITCH_REQ;

            break;

        // 上記以外の場合、以下を実行する。
        default:

            resTelegramId = reqTelegramId;

            break;

        }

        return resTelegramId;
    }
}
