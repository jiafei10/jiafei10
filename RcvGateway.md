 * LscRcvGateway.java.
 *
 * All rights reserved, Copyright(C) 2023 Oki Electric Industry Co.,Ltd.
 */

package com.oki.ofits.common.api.extif.lsc;

import java.io.UnsupportedEncodingException;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

import javax.validation.ConstraintViolation;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

import com.oki.ofits.common.api.extif.UdpGateway;
import com.oki.ofits.common.constant.Const;
import com.oki.ofits.common.constant.SystemCode;
import com.oki.ofits.common.controlparam.SystemControlParam.IfLscControlParam;
import com.oki.ofits.common.logging.AppLogger;
import com.oki.ofits.common.message.MsgId;
import com.oki.ofits.logic.model.extif.MessageResultDto;
import com.oki.ofits.logic.model.extif.lsc.LscBaseStaChStatusNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscBaseStaIncallNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscBaseStaSelectStatusResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscChOccCommConfResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscComHeaderDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommConfResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommEndResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommStartEndNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommStartResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscFailerNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscMaintenanceCallStatusNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscMobileStaAreaInfoGetNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscOpeStartNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscOutcallerNoNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscReglationCtrlStatusResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscSelcallCallNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscSelcallCommNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscSelcallResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscSwitchNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscTimeConfChangeResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscTimeNoticeRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscVersionNoticeRcvDto;

/**
 * LAN信号変換装置(通知受信)Gatewayクラス.
 * <p>
 * 
 *
 *  LAN信号変換装置から電文を受信した時に動作し、各Controllerに通知する。 
 * </p>
 */
@Component
public class LscRcvGateway extends UdpGateway {

    /** パブリッシャー. */
    private ApplicationEventPublisher publisher;

    /** アプリケーションロガー. */
    private AppLogger logger;

    /** LAN信号変換装置要求応答レシーバー. */
    private LscReqResReceiver lscReqResReceiver;

    @Autowired
    public LscRcvGateway(ApplicationEventPublisher publisher, AppLogger logger, LscReqResReceiver lscReqResReceiver) {
        this.publisher = publisher;
        this.logger = logger;
        this.lscReqResReceiver = lscReqResReceiver;
    }

    /**
     * データ受信メソッド
     * 
     * LAN信号変換装置から受信したデータを変換し、Controllerに通知する。
     * @param rcvData 受信データ
     * @return void
     */
    public void rcvData(byte[] rcvData) {

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文データ出力】", String.format("LAN信号変換装置(%s)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("rcvData", this.shapingByteList(rcvData));

        logger.debug(callerClazz, "", debugInfo);

        //1. 以下の自クラスメソッドを呼び出す。
        LscComHeaderDto comHeaderDto = null;
        try {
            comHeaderDto = convLscComHeaderToObject(rcvData);

        } catch (Exception e) {

            logger.fatal(callerClazz, MsgId.FAE_04_00012,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LSC共通ヘッダ", "LSC共通ヘッダDTO"),
                    debugInfo, e);

            return;
        }

        //2. LSC共通ヘッダDTO.電文セパレータ[comHeaderDto.telegramSeparator]の値に応じて、以下の処理をする。

        if (Objects.isNull(comHeaderDto.telegramSeparator)) {
            //2.1. LSC共通ヘッダDTO.電文セパレータ[comHeaderDto.telegramSeparator] = NULLの場合、以下の処理をする。

            //2.1.1. ワーニングログを出力する。
            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2. 上記以外の場合、以下を処理する。
        } else {

            MessageResultDto messageResultDto = null;
            LscResRcvDataDto lscResRcvDataDto = null;

            //2.2.1. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]の値により以下を実行する。

            //2.2.1.1. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550461：共通定数.外部IF(LAN信号変換装置).電文識別(時刻要求) [Const.IF_LSC_TELE_ID_TIME_REQ]"の場合、以下を実行する。
            switch (comHeaderDto.telegramIdent) {
            case Const.IF_LSC_TELE_ID_TIME_REQ:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 時刻要求(LSC時間通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.1.1. 以下の自クラスモジュールを呼び出す。
                LscTimeNoticeRcvDto timeNoticeRcvDto = convTimeToObject(rcvData, comHeaderDto);

                //2.2.1.1.3. ControllerにLSC時間通知受信DTOを通知する。

                //2.2.1.1.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(timeNoticeRcvDto);

                break;

            //2.2.1.2. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550464：共通定数.外部IF(LAN信号変換装置).電文識別(基地局着信通知) [Const.IF_LSC_TELE_ID_BASE_STA_INCALL_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_BASE_STA_INCALL_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 基地局着信通知(LSC基地局着信通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.2.1. 以下の自クラスモジュールを呼び出す。
                LscBaseStaIncallNoticeRcvDto baseStaIncallNoticeRcvDto =
                        convBaseStaIncallToObject(rcvData, comHeaderDto);

                //2.2.1.2.3. ControllerにLSC基地局着信通知受信DTOを通知する。

                //2.2.1.2.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(baseStaIncallNoticeRcvDto);

                break;

            //2.2.1.3. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550465：共通定数.外部IF(LAN信号変換装置).電文識別(障害通知) [Const.IF_LSC_TELE_ID_INCIDENT_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_INCIDENT_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 障害通知(LSC障害通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.3.1. 以下の自クラスモジュールを呼び出す。
                LscFailerNoticeRcvDto lscFailerNoticeRcvDto = convLscFailerToObject(rcvData, comHeaderDto);

                //2.2.1.3.3. ControllerにLSC障害通知受信DTOを通知する。

                //2.2.1.3.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(lscFailerNoticeRcvDto);

                break;

            //2.2.1.4. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550468：共通定数.外部IF(LAN信号変換装置).電文識別(セレコール呼出通知) [Const.IF_LSC_TELE_ID_SELECALL_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_SELECALL_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - セレコール呼出通知(LSCセレコール呼出通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.4.1. 以下の自クラスモジュールを呼び出す。
                LscSelcallCallNoticeRcvDto selcallCallNoticeRcvDto = convSelcallCallToObject(rcvData, comHeaderDto);

                //2.2.1.4.3. ControllerにLSCセレコール呼出通知受信DTOを通知する。

                //2.2.1.4.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(selcallCallNoticeRcvDto);

                break;

            //2.2.1.5. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550471：共通定数.外部IF(LAN信号変換装置).電文識別(運用開始要求) [Const.IF_LSC_TELE_ID_OPE_START_REQ]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_OPE_START_REQ:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 運用開始要求(LSC運用開始通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.5.1. 以下の自クラスモジュールを呼び出す。
                LscOpeStartNoticeRcvDto opeStartNoticeRcvDto = convOpeStartToObject(rcvData, comHeaderDto);

                //2.2.1.5.3. ControllerにLSC運用開始通知受信DTOを通知する。

                //2.2.1.5.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(opeStartNoticeRcvDto);

                break;

            //2.2.1.6. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550473：共通定数.外部IF(LAN信号変換装置).電文識別(通信開始/終了通知) [Const.IF_LSC_TELE_ID_TELECOM_START_END_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_TELECOM_START_END_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 通信開始/終了通知(LSC通信開始終了通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.6.1. 以下の自クラスモジュールを呼び出す。
                LscCommStartEndNoticeRcvDto lscCommStartEndNoticeRcvDto =
                        convLscCommStartEndToObject(rcvData, comHeaderDto);

                //2.2.1.6.3. ControllerにLSC通信開始終了通知受信DTOを通知する。

                //2.2.1.6.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(lscCommStartEndNoticeRcvDto);

                break;

            //2.2.1.7. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550474：共通定数.外部IF(LAN信号変換装置).電文識別(規制制御状態通知) [Const.IF_LSC_TELE_ID_REGULATION_STATUS_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_REGULATION_STATUS_NOTICE:

                // LscGateway.sndReglationCtrlStatusReq/convReglationCtrlStatusToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 規制制御状態通知(LSC規制制御状態応答受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscReglationCtrlStatusResRcvDto lscReglationCtrlStatusResRcvDto =
                        convReglationCtrlStatusResToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscReglationCtrlStatusResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscReglationCtrlStatusResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscReglationCtrlStatusResRcvDto.chCode;
                boolean reglationCtrlStatusResRcvFlag =
                        lscReqResReceiver.setResponse(lscResRcvDataDto);

                // 応答電文ではなかった場合、通知電文として扱う
                if (!reglationCtrlStatusResRcvFlag) {
                    //2.2.1.7.1. 以下の自クラスモジュールを呼び出す。
                    LscReglationCtrlStatusResRcvDto reglationCtrlStatusResRcvDto =
                            convReglationCtrlStatusToObject(rcvData, comHeaderDto);

                    //2.2.1.7.3. ControllerにLSC規制制御状態応答受信DTOを通知する。

                    //2.2.1.7.3. 以下のJava標準モジュールを呼び出す。
                    publisher.publishEvent(reglationCtrlStatusResRcvDto);
                }

                break;

            //2.2.1.8. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550476：共通定数.外部IF(LAN信号変換装置).電文識別(基地局CH状態通知) [Const.IF_LSC_TELE_ID_BASE_STA_CH_STATUS_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_BASE_STA_CH_STATUS_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 基地局CH状態通知(LSC基地局CH状態通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.8.1. 以下の自クラスモジュールを呼び出す。
                LscBaseStaChStatusNoticeRcvDto baseStaChStatusNoticeRcvDto =
                        convBaseStaChStatusToObject(rcvData, comHeaderDto);

                //2.2.1.8.3. ControllerにLSC基地局CH状態通知受信DTOを通知する。

                //2.2.1.8.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(baseStaChStatusNoticeRcvDto);

                break;

            //2.2.1.9. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550477：共通定数.外部IF(LAN信号変換装置).電文識別(発信者番号通知) [Const.IF_LSC_TELE_ID_INCALLER_NO_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_INCALL_NO_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 発信者番号通知(LSC発信者番号通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.9.1. 以下の自クラスモジュールを呼び出す。
                LscOutcallerNoNoticeRcvDto outcallerNoNoticeRcvDto = convOutcallerNoToObject(rcvData, comHeaderDto);

                //2.2.1.9.3. ControllerにLSC発信者番号通知受信DTOを通知する。

                //2.2.1.9.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(outcallerNoNoticeRcvDto);

                break;

            //2.2.1.10. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550480：共通定数.外部IF(LAN信号変換装置).電文識別(セレコール通信応答通知) [Const.IF_LSC_TELE_ID_SELECALL_TELECOM_RES_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_SELECALL_TELECOM_RES_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - セレコール通信応答通知(LSCセレコール通信通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.10.1. 以下の自クラスモジュールを呼び出す。
                LscSelcallCommNoticeRcvDto selcallCommNoticeRcvDto = convSelcallCommToObject(rcvData, comHeaderDto);

                //2.2.1.10.3. ControllerにLSCセレコール通信通知受信DTOを通知する。

                //2.2.1.10.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(selcallCommNoticeRcvDto);

                break;

            //2.2.1.11. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800113：共通定数.外部IF(LAN信号変換装置).電文識別(移動局在圏情報取得要求) [Const.IF_LSC_TELE_ID_MOBILE_STA_AREA_REQ]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_MOBILE_STA_AREA_REQ:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 移動局在圏情報取得応答(LSC移動局在圏情報取得通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.11.1. 以下の自クラスモジュールを呼び出す。
                LscMobileStaAreaInfoGetNoticeRcvDto mobileStaAreaInfoGetNoticeRcvDto =
                        convMobileStaAreaInfoGetToObject(rcvData, comHeaderDto);

                //2.2.1.11.3. ControllerにLSC移動局在圏情報取得通知受信DTOを通知する。

                //2.2.1.11.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(mobileStaAreaInfoGetNoticeRcvDto);

                break;

            //2.2.1.12. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800114：共通定数.外部IF(LAN信号変換装置).電文識別(バージョン要求) [Const.IF_LSC_TELE_ID_VER_REQ]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_VER_REQ:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - バージョン通知(LSCバージョン通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.12.1. 以下の自クラスモジュールを呼び出す。
                LscVersionNoticeRcvDto versionNoticeRcvDto = convVersionToObject(rcvData, comHeaderDto);

                //2.2.1.12.3. ControllerにLSCバージョン通知受信DTOを通知する。

                //2.2.1.12.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(versionNoticeRcvDto);

                break;

            //2.2.1.13. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800115：共通定数.外部IF(LAN信号変換装置).電文識別(LAN信号変換装置系切替要求) [Const.IF_LSC_TELE_ID_LAN_SWITCH_REQ]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_LAN_SWITCH_REQ:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - LAN信号変換装置系切替応答(LSC切替通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.13.1. 以下の自クラスモジュールを呼び出す。
                LscSwitchNoticeRcvDto lscSwitchNoticeRcvDto = convLscSwitchToObject(rcvData, comHeaderDto);

                //2.2.1.13.3. ControllerにLAN信号変換装置LSC切替通知受信DTOを通知する。

                //2.2.1.13.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(lscSwitchNoticeRcvDto);

                break;

            //2.2.1.14. LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800209：共通定数.外部IF(LAN信号変換装置).電文識別(保守通話状態通知) [Const.IF_LSC_TELE_ID_MAINTENANCE_CALL_STATUS_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_MAINTENANCE_CALL_STATUS_NOTICE:

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 保守通話状態通知(LSC保守通話状態通知受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                //2.2.1.14.1. 以下の自クラスモジュールを呼び出す。
                LscMaintenanceCallStatusNoticeRcvDto maintenanceCallStatusNoticeRcvDto =
                        convMaintenanceCallStatusToObject(rcvData, comHeaderDto);

                //2.2.1.14.3. ControllerにLSC保守通話状態通知受信DTOを通知する。

                //2.2.1.14.3. 以下のJava標準モジュールを呼び出す。
                publisher.publishEvent(maintenanceCallStatusNoticeRcvDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550463：共通定数.外部IF(LAN信号変換装置).電文識別(基地局選択状態通知) [Const.IF_LSC_TELE_ID_BASE_STA_SELECT_STATUS_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_BASE_STA_SELECT_STATUS_NOTICE:

                // LscGateway.sndBaseStaSelectStatusReq/convBaseStaSelectStatusNoticeToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 基地局選択状態通知(LSC基地局選択状態応答受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscBaseStaSelectStatusResRcvDto lscBaseStaSelectStatusResRcvDto =
                        convBaseStaSelectStatusNoticeResToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscBaseStaSelectStatusResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscBaseStaSelectStatusResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscBaseStaSelectStatusResRcvDto.chCode;
                boolean baseStaSelectStatusResRcvFlag =
                        lscReqResReceiver.setResponse(lscResRcvDataDto);

                // 応答電文ではなかった場合、通知電文として扱う
                if (!baseStaSelectStatusResRcvFlag) {
                    //2.2.1.7.1. 以下の自クラスモジュールを呼び出す。
                    LscBaseStaSelectStatusResRcvDto baseStaSelectStatusResRcvDto =
                            convBaseStaSelectStatusNoticeToObject(rcvData, comHeaderDto);

                    //2.2.1.7.3. ControllerにLSC規制制御状態応答受信DTOを通知する。

                    //2.2.1.7.3. 以下のJava標準モジュールを呼び出す。
                    publisher.publishEvent(baseStaSelectStatusResRcvDto);
                }

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550472：共通定数.外部IF(LAN信号変換装置).電文識別(通信設定応答通知) [Const.IF_LSC_TELE_ID_TELECOM_CONF_RES_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_TELECOM_CONF_RES_NOTICE:

                // LscGateway.sndLscCommConfReq/convCommConfResNoticeToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 通信設定応答通知(LSC通信設定応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscCommConfResRcvDto lscCommConfResRcvDto =
                        convCommConfResNoticeToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscCommConfResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscCommConfResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscCommConfResRcvDto.chCode;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550478：共通定数.外部IF(LAN信号変換装置).電文識別(通信開始応答通知) [Const.IF_LSC_TELE_ID_TELECOM_START_RES_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_TELECOM_START_RES_NOTICE:

                // LscGateway.sndLscCommStartReq/convCommStartResNoticeToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 通信開始応答通知(LSC通信開始応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscCommStartResRcvDto lscCommStartResRcvDto =
                        convCommStartResNoticeToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscCommStartResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscCommStartResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscCommStartResRcvDto.chCode;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550479：共通定数.外部IF(LAN信号変換装置).電文識別(通信終了応答通知) [Const.IF_LSC_TELE_ID_TELECOM_END_RES_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_TELECOM_END_RES_NOTICE:

                // LscGateway.sndLscCommEndReq/convCommEndResNoticeToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 通信終了応答通知(LSC通信終了応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscCommEndResRcvDto lscCommEndResRcvDto =
                        convCommEndResNoticeToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscCommEndResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscCommEndResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscCommEndResRcvDto.chCode;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"550469：共通定数.外部IF(LAN信号変換装置).電文識別(セレコール応答受信通知) [Const.IF_LSC_TELE_ID_SELECALL_RES_RECV_NOTICE]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_SELECALL_RES_RECV_NOTICE:

                // LscGateway.sndSelcallReq/convSelcallResRcvNoticeToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - セレコール応答受信通知(LSCセレコール応答受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscSelcallResRcvDto lscSelcallResRcvDto =
                        convSelcallResRcvNoticeToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscSelcallResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscSelcallResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = lscSelcallResRcvDto.chCode;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800103：共通定数.外部IF(LAN信号変換装置).電文識別(チャネル占有通信設定応答) [Const.IF_LSC_TELE_ID_CHANNEL_TELECOM_CONF_RES]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_CHANNEL_TELECOM_CONF_RES:

                // LscGateway.sndChOccCommConfReq/convChOccCommConfResToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - チャネル占有通信設定応答(LSCチャネル占有通信設定応答受信DTO)",
                                IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscChOccCommConfResRcvDto chOccLscCommConfResRcvDto =
                        convChOccCommConfResToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = chOccLscCommConfResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = chOccLscCommConfResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = null;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            // LSC共通ヘッダDTO.電文識別[comHeaderDto.telegramIdent]が"800101：共通定数.外部IF(LAN信号変換装置).電文識別(時刻設定変更応答) [Const.IF_LSC_TELE_ID_TIME_CONF_CHANGE_RES]"の場合、以下を実行する。
            case Const.IF_LSC_TELE_ID_TIME_CONF_CHANGE_RES:

                // LscGateway.sndTimeConfChangeReq/convTimeConfChangeResToObject

                // 【ログ出力】
                // デバッグログを出力
                debugInfo = new LinkedHashMap<>();
                debugInfo.put("【受信電文データ出力】",
                        String.format("LAN信号変換装置(%s) - 時刻設定変更応答(LSC時間設定変更応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
                debugInfo.put("rcvData", this.shapingByteList(rcvData));

                logger.debug(callerClazz, "", debugInfo);

                // 以下の自クラスモジュールを呼び出す。
                messageResultDto = new MessageResultDto();
                LscTimeConfChangeResRcvDto lscTimeConfChangeResRcvDto =
                        convTimeConfChangeResToObject(rcvData, comHeaderDto, messageResultDto);

                // 応答電文を設定する。
                lscResRcvDataDto = new LscResRcvDataDto();
                lscResRcvDataDto.messageResultDto = messageResultDto;
                lscResRcvDataDto.rcvData = lscTimeConfChangeResRcvDto;
                lscResRcvDataDto.rcvDataLscComHeaderDto = lscTimeConfChangeResRcvDto.comHeaderDto;
                lscResRcvDataDto.chCode = null;
                lscReqResReceiver.setResponse(lscResRcvDataDto);

                break;

            //2.2.15. 上記以外の場合、以下を実行する。
            default:

                break;

            //2.2.15.1. ワーニングログを出力する。

            }
        }

        //3. 処理を終了する。

    }

    /**
     * オブジェクト変換(共通ヘッダ)メソッド
     * 
     * LAN信号変換装置から受信したデータをLSC共通ヘッダに変換する。
     * @param rcvData 受信データ
     * @return comHeaderDto LSC共通ヘッダDTO
     */
    private LscComHeaderDto convLscComHeaderToObject(byte[] rcvData) {

        //1. LSC共通ヘッダDTO[lscComHeaderDto]を生成する。
        LscComHeaderDto lscComHeaderDto = new LscComHeaderDto();

        try {

            //2. LSC共通ヘッダDTO[lscComHeaderDto]に以下の値を設定する。
            lscComHeaderDto.telegramSeparator = new String(Arrays.copyOfRange(rcvData, 0, 2), "MS932");
            lscComHeaderDto.telegramLength = new String(Arrays.copyOfRange(rcvData, 2, 6), "MS932");
            lscComHeaderDto.groupCode = new String(Arrays.copyOfRange(rcvData, 6, 10), "MS932");
            lscComHeaderDto.telegramIdent = new String(Arrays.copyOfRange(rcvData, 10, 16), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        //3. 戻り値としてLSC共通ヘッダDTO[lscComHeaderDto]を返却して、処理を終了する。
        return lscComHeaderDto;

    }

    /**
     * オブジェクト変換(時刻要求)メソッド
     * 
     * LAN信号変換装置から受信したデータを時刻要求に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscTimeNoticeRcvDto LSC時間通知受信DTO
     */
    private LscTimeNoticeRcvDto convTimeToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC時間通知受信DTO[timeNoticeRcvDto]を生成する。
        LscTimeNoticeRcvDto timeNoticeRcvDto = new LscTimeNoticeRcvDto();

        //2. LSC時間通知受信DTO[timeNoticeRcvDto]に以下の値を設定する。
        timeNoticeRcvDto.comHeaderDto = comHeaderDto;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 時刻要求(LSC時間通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("timeNoticeRcvDto", timeNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //3. データ検証を行う。

        //3.1. LSC時間通知受信DTO[timeNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscTimeNoticeRcvDto>> checkResult = this.validateData(timeNoticeRcvDto);

        //3.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //3.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscTimeNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "時刻要求", "LSC時間通知受信DTO"), debugInfo);

        }

        //5. 戻り値としてLSC時間通知受信DTO[timeNoticeRcvDto]を返却し、処理を終了する。
        return timeNoticeRcvDto;

    }

    /**
     * オブジェクト変換(基地局着信通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを基地局着信通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscBaseStaIncallNoticeRcvDto LSC基地局着信通知受信DTO
     */
    private LscBaseStaIncallNoticeRcvDto convBaseStaIncallToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC基地局着信通知受信DTO[baseStaIncallNoticeRcvDto]を生成する。
        LscBaseStaIncallNoticeRcvDto baseStaIncallNoticeRcvDto = new LscBaseStaIncallNoticeRcvDto();

        //2. LSC基地局着信通知受信DTO[baseStaIncallNoticeRcvDto]に以下の値を設定する。
        baseStaIncallNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {
            //3. LSC基地局着信通知受信DTO[baseStaIncallNoticeRcvDto]に以下の値を設定する。
            baseStaIncallNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            baseStaIncallNoticeRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            baseStaIncallNoticeRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");
            baseStaIncallNoticeRcvDto.outcallSrcNo = Arrays.copyOfRange(rcvData, 23, 26);
            baseStaIncallNoticeRcvDto.order = new String(Arrays.copyOfRange(rcvData, 26, 27), "MS932");

        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 基地局着信通知(LSC基地局着信通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("baseStaIncallNoticeRcvDto", baseStaIncallNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC基地局着信通知受信DTO[baseStaIncallNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscBaseStaIncallNoticeRcvDto>> checkResult =
                this.validateData(baseStaIncallNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscBaseStaIncallNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(),
                        checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "基地局着信通知", "LSC基地局着信通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC基地局着信通知受信DTO[baseStaIncallNoticeRcvDto]を返却し、処理を終了する。
        return baseStaIncallNoticeRcvDto;

    }

    /**
     * オブジェクト変換(障害通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを障害通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscFailerNoticeRcvDto LSC障害通知受信DTO
     */
    private LscFailerNoticeRcvDto convLscFailerToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC障害通知受信DTO[lscFailerNoticeRcvDto]を生成する。
        LscFailerNoticeRcvDto lscFailerNoticeRcvDto = new LscFailerNoticeRcvDto();

        //2. LSC障害通知受信DTO[lscFailerNoticeRcvDto]に以下の値を設定する。
        lscFailerNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC障害通知受信DTO[lscFailerNoticeRcvDto]に以下の値を設定する。
            lscFailerNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            lscFailerNoticeRcvDto.order = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 障害通知(LSC障害通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscFailerNoticeRcvDto", lscFailerNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC障害通知受信DTO[lscFailerNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscFailerNoticeRcvDto>> checkResult = this.validateData(lscFailerNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscFailerNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "障害通知", "LSC障害通知受信DTO"), debugInfo);

        }

        //5. 戻り値としてLSC障害通知受信DTO[lscFailerNoticeRcvDto]を返却し、処理を終了する。
        return lscFailerNoticeRcvDto;

    }

    /**
     * オブジェクト変換(セレコール呼出通知)メソッド
     * 
     * LAN信号変換装置から受信したデータをセレコール呼出通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscSelcallCallNoticeRcvDto LSCセレコール呼出通知受信DTO
     */
    private LscSelcallCallNoticeRcvDto convSelcallCallToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSCセレコール呼出通知受信DTO[selcallCallNoticeRcvDto]を生成する。
        LscSelcallCallNoticeRcvDto selcallCallNoticeRcvDto = new LscSelcallCallNoticeRcvDto();

        //2. LSCセレコール呼出通知受信DTO[selcallCallNoticeRcvDto]に以下の値を設定する。
        selcallCallNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSCセレコール呼出通知受信DTO[selcallCallNoticeRcvDto]に以下の値を設定する。
            selcallCallNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            selcallCallNoticeRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");
            selcallCallNoticeRcvDto.outcallSrcNoLength = new String(Arrays.copyOfRange(rcvData, 20, 21), "MS932");
            selcallCallNoticeRcvDto.outcallSrcNo = new String(Arrays.copyOfRange(rcvData, 21, 29), "MS932");
            selcallCallNoticeRcvDto.incallDestNoLength = new String(Arrays.copyOfRange(rcvData, 29, 30), "MS932");
            selcallCallNoticeRcvDto.incallDestNo = new String(Arrays.copyOfRange(rcvData, 30, 38), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - セレコール呼出通知(LSCセレコール呼出通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("selcallCallNoticeRcvDto", selcallCallNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSCセレコール呼出通知受信DTO[selcallCallNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSelcallCallNoticeRcvDto>> checkResult = this.validateData(selcallCallNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSelcallCallNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "セレコール呼出通知", "LSCセレコール呼出通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSCセレコール呼出通知受信DTO[selcallCallNoticeRcvDto]を返却し、処理を終了する。
        return selcallCallNoticeRcvDto;

    }

    /**
     * オブジェクト変換(運用開始要求)メソッド
     * 
     * LAN信号変換装置から受信したデータを運用開始要求に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscOpeStartNoticeRcvDto LSC運用開始通知受信DTO
     */
    private LscOpeStartNoticeRcvDto convOpeStartToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC運用開始通知受信DTO[opeStartNoticeRcvDto]を生成する。
        LscOpeStartNoticeRcvDto opeStartNoticeRcvDto = new LscOpeStartNoticeRcvDto();

        //2. LSC運用開始通知受信DTO[opeStartNoticeRcvDto]に以下の値を設定する。
        opeStartNoticeRcvDto.comHeaderDto = comHeaderDto;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 運用開始要求(LSC運用開始通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("opeStartNoticeRcvDto", opeStartNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //3. データ検証を行う。

        //3.1. LSC運用開始通知受信DTO[opeStartNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscOpeStartNoticeRcvDto>> checkResult = this.validateData(opeStartNoticeRcvDto);

        //3.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //3.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //3.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscOpeStartNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "運用開始要求", "LSC運用開始通知受信DTO"),
                    debugInfo);

        }

        //4. 戻り値としてLSC運用開始通知受信DTO[opeStartNoticeRcvDto]を返却し、処理を終了する。
        return opeStartNoticeRcvDto;

    }

    /**
     * オブジェクト変換(通信開始-終了通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを通信開始-終了通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscCommStartEndNoticeRcvDto LSC通信開始終了通知受信DTO
     */
    private LscCommStartEndNoticeRcvDto convLscCommStartEndToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC通信開始終了通知受信DTO[lscCommStartEndNoticeRcvDto]を生成する。
        LscCommStartEndNoticeRcvDto lscCommStartEndNoticeRcvDto = new LscCommStartEndNoticeRcvDto();

        //2. LSC通信開始終了通知受信DTO[lscCommStartEndNoticeRcvDto]に以下の値を設定する。
        lscCommStartEndNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC通信開始終了通知受信DTO[lscCommStartEndNoticeRcvDto]に以下の値を設定する。
            lscCommStartEndNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            lscCommStartEndNoticeRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            lscCommStartEndNoticeRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");
            lscCommStartEndNoticeRcvDto.transmissionType = new String(Arrays.copyOfRange(rcvData, 23, 24), "MS932");
            lscCommStartEndNoticeRcvDto.direction = new String(Arrays.copyOfRange(rcvData, 24, 25), "MS932");
            lscCommStartEndNoticeRcvDto.startEnd = new String(Arrays.copyOfRange(rcvData, 25, 26), "MS932");
            lscCommStartEndNoticeRcvDto.disconnectReason = new String(Arrays.copyOfRange(rcvData, 26, 28), "MS932");
            lscCommStartEndNoticeRcvDto.outcallSrcNoLength = new String(Arrays.copyOfRange(rcvData, 28, 29), "MS932");
            lscCommStartEndNoticeRcvDto.outcallSrcNo = new String(Arrays.copyOfRange(rcvData, 29, 37), "MS932");
            lscCommStartEndNoticeRcvDto.incallDestNoLength = new String(Arrays.copyOfRange(rcvData, 37, 38), "MS932");
            lscCommStartEndNoticeRcvDto.incallDestNoNo = new String(Arrays.copyOfRange(rcvData, 38, 46), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 通信開始/終了通知(LSC通信開始終了通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommStartEndNoticeRcvDto", lscCommStartEndNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC通信開始終了通知受信DTO[lscCommStartEndNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommStartEndNoticeRcvDto>> checkResult =
                this.validateData(lscCommStartEndNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommStartEndNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "通信開始/終了通知", "LSC通信開始終了通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC通信開始終了通知受信DTO[lscCommStartEndNoticeRcvDto]を返却し、処理を終了する。
        return lscCommStartEndNoticeRcvDto;

    }

    /**
     * オブジェクト変換(規制制御状態通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを規制制御状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscReglationCtrlStatusResRcvDto LSC規制制御状態応答受信DTO
     */
    private LscReglationCtrlStatusResRcvDto convReglationCtrlStatusToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]を生成する。
        LscReglationCtrlStatusResRcvDto reglationCtrlStatusResRcvDto = new LscReglationCtrlStatusResRcvDto();

        //2. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に以下の値を設定する。
        reglationCtrlStatusResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に以下の値を設定する。
            reglationCtrlStatusResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            reglationCtrlStatusResRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        //4. SACCH/FACCH情報を設定する。

        //4.1. SACCH/FACCH情報[reglationCtrlStatusResRcvDto.ctrlChInfo]を生成する。
        reglationCtrlStatusResRcvDto.ctrlChInfo = new LscReglationCtrlStatusResRcvDto.CtrlChInfo();

        try {

            //4.2. SACCH/FACCH情報[reglationCtrlStatusResRcvDto.ctrlChInfo]に以下の値を設定する。
            reglationCtrlStatusResRcvDto.ctrlChInfo.messageType = Byte.toUnsignedInt(rcvData[22]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.reglationInfo = Byte.toUnsignedInt(rcvData[23]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.dispatchCmdDetail = Byte.toUnsignedInt(rcvData[24]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.commReglationDetail = Byte.toUnsignedInt(rcvData[25]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.emergSignalDetail = Byte.toUnsignedInt(rcvData[26]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.forcedDisconnectDetail = Byte.toUnsignedInt(rcvData[27]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.userData =
                    new String(
                            Arrays.copyOfRange(rcvData, 28,
                                    Integer.parseInt(reglationCtrlStatusResRcvDto.comHeaderDto.telegramLength)),
                            "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 規制制御状態通知(LSC規制制御状態応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("reglationCtrlStatusResRcvDto", reglationCtrlStatusResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //5. データ検証を行う。

        //5.1. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscReglationCtrlStatusResRcvDto>> checkResult =
                this.validateData(reglationCtrlStatusResRcvDto);

        //5.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //5.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //5.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscReglationCtrlStatusResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "規制制御状態通知", "LSC規制制御状態応答受信DTO"),
                    debugInfo);

            //5.2.2. 上記以外の場合、何もしない。
        } else {}

        //6. 戻り値としてLSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]を返却し、処理を終了する。
        return reglationCtrlStatusResRcvDto;

    }

    /**
     * オブジェクト変換(基地局CH 状態通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを基地局CH 状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscBaseStaChStatusNoticeRcvDto LSC基地局CH状態通知受信DTO
     */
    private LscBaseStaChStatusNoticeRcvDto convBaseStaChStatusToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC基地局CH状態通知受信DTO[baseStaChStatusNoticeRcvDto]を生成する。
        LscBaseStaChStatusNoticeRcvDto baseStaChStatusNoticeRcvDto = new LscBaseStaChStatusNoticeRcvDto();

        //2. LSC基地局CH状態通知受信DTO[baseStaChStatusNoticeRcvDto]に以下の値を設定する。
        baseStaChStatusNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC基地局CH状態通知受信DTO[baseStaChStatusNoticeRcvDto]に以下の値を設定する。
            baseStaChStatusNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            baseStaChStatusNoticeRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            baseStaChStatusNoticeRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 基地局CH状態通知(LSC基地局CH状態通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("baseStaChStatusNoticeRcvDto", baseStaChStatusNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscBaseStaChStatusNoticeRcvDto>> checkResult =
                this.validateData(baseStaChStatusNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscBaseStaChStatusNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "基地局CH状態通知", "LSC基地局CH状態通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]を返却し、処理を終了する。
        return baseStaChStatusNoticeRcvDto;

    }

    /**
     * オブジェクト変換(発信者番号通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを発信者番号通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscOutcallerNoNoticeRcvDto LSC発信者番号通知受信DTO
     */
    private LscOutcallerNoNoticeRcvDto convOutcallerNoToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]を生成する。
        LscOutcallerNoNoticeRcvDto outcallerNoNoticeRcvDto = new LscOutcallerNoNoticeRcvDto();

        //2. LSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]に以下の値を設定する。
        outcallerNoNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]に以下の値を設定する。
            outcallerNoNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            outcallerNoNoticeRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            outcallerNoNoticeRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");
            outcallerNoNoticeRcvDto.direction = new String(Arrays.copyOfRange(rcvData, 23, 24), "MS932");
            outcallerNoNoticeRcvDto.incallSrcNo = Arrays.copyOfRange(rcvData, 24, 27);
            outcallerNoNoticeRcvDto.order = new String(Arrays.copyOfRange(rcvData, 27, 28), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 発信者番号通知(LSC発信者番号通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("outcallerNoNoticeRcvDto", outcallerNoNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscOutcallerNoNoticeRcvDto>> checkResult = this.validateData(outcallerNoNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscOutcallerNoNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "発信者番号通知", "LSC発信者番号通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC発信者番号通知受信DTO[outcallerNoNoticeRcvDto]を返却し、処理を終了する。
        return outcallerNoNoticeRcvDto;

    }

    /**
     * オブジェクト変換(セレコール通信応答通知)メソッド
     * 
     * LAN信号変換装置から受信したデータをセレコール通信応答通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscSelcallCommNoticeRcvDto LSCセレコール通信通知受信DTO
     */
    private LscSelcallCommNoticeRcvDto convSelcallCommToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSCセレコール通信通知受信DTO[selcallCommNoticeRcvDto]を生成する。
        LscSelcallCommNoticeRcvDto selcallCommNoticeRcvDto = new LscSelcallCommNoticeRcvDto();

        //2. LSCセレコール通信通知受信DTO[selcallCommNoticeRcvDto]に以下の値を設定する。
        selcallCommNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSCセレコール通信通知受信DTO[selcallCommNoticeRcvDto]に以下の値を設定する。
            selcallCommNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            selcallCommNoticeRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");
            selcallCommNoticeRcvDto.resRslt = new String(Arrays.copyOfRange(rcvData, 20, 21), "MS932");
            selcallCommNoticeRcvDto.incallDestDestNoLength = new String(Arrays.copyOfRange(rcvData, 21, 22), "MS932");
            selcallCommNoticeRcvDto.incallDestDestNo = new String(Arrays.copyOfRange(rcvData, 22, 30), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - セレコール通信応答通知(LSCセレコール通信通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("selcallCommNoticeRcvDto", selcallCommNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSCセレコール通信通知受信DTO[selcallCommNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSelcallCommNoticeRcvDto>> checkResult = this.validateData(selcallCommNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSelcallCommNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "セレコール通信応答通知", "LSCセレコール通信通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSCセレコール通信通知受信DTO[selcallCommNoticeRcvDto]を返却し、処理を終了する。
        return selcallCommNoticeRcvDto;

    }

    /**
     * オブジェクト変換(移動局在圏情報取得要求)メソッド
     * 
     * LAN信号変換装置から受信したデータを移動局在圏情報取得要求に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscMobileStaAreaInfoGetNoticeRcvDto LSC移動局在圏情報取得通知受信DTO
     */
    private LscMobileStaAreaInfoGetNoticeRcvDto convMobileStaAreaInfoGetToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC移動局在圏情報取得通知受信DTO[mobileStaAreaInfoGetNoticeRcvDto]を生成する。
        LscMobileStaAreaInfoGetNoticeRcvDto mobileStaAreaInfoGetNoticeRcvDto =
                new LscMobileStaAreaInfoGetNoticeRcvDto();

        //2. LSC移動局在圏情報取得通知受信DTO[mobileStaAreaInfoGetNoticeRcvDto]に以下の値を設定する。
        mobileStaAreaInfoGetNoticeRcvDto.comHeaderDto = comHeaderDto;

        //3. LSC移動局在圏情報取得通知受信DTO[mobileStaAreaInfoGetNoticeRcvDto]に以下の値を設定する。
        try {
            mobileStaAreaInfoGetNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            mobileStaAreaInfoGetNoticeRcvDto.mobileStaBaseNo = Arrays.copyOfRange(rcvData, 19, 22);
        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 移動局在圏情報取得応答(LSC移動局在圏情報取得通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("mobileStaAreaInfoGetNoticeRcvDto", mobileStaAreaInfoGetNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC移動局在圏情報取得通知受信DTO[mobileStaAreaInfoGetNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscMobileStaAreaInfoGetNoticeRcvDto>> checkResult =
                this.validateData(mobileStaAreaInfoGetNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscMobileStaAreaInfoGetNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "移動局在圏情報取得応答", "LSC移動局在圏情報取得通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC移動局在圏情報取得通知受信DTO[mobileStaAreaInfoGetNoticeRcvDto]を返却し、処理を終了する。
        return mobileStaAreaInfoGetNoticeRcvDto;

    }

    /**
     * オブジェクト変換(バージョン要求)メソッド
     * 
     * LAN信号変換装置から受信したデータをバージョン要求に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscVersionNoticeRcvDto LSCバージョン通知受信DTO
     */
    private LscVersionNoticeRcvDto convVersionToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSCバージョン通知受信DTO[versionNoticeRcvDto]を生成する。
        LscVersionNoticeRcvDto versionNoticeRcvDto = new LscVersionNoticeRcvDto();

        //2. LSCバージョン通知受信DTO[versionNoticeRcvDto]に以下の値を設定する。
        versionNoticeRcvDto.comHeaderDto = comHeaderDto;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - バージョン通知(LSCバージョン通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("versionNoticeRcvDto", versionNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //3. データ検証を行う。

        //3.1. LSCバージョン通知受信DTO[versionNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscVersionNoticeRcvDto>> checkResult = this.validateData(versionNoticeRcvDto);

        //3.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //3.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscVersionNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "バージョン通知", "LSCバージョン通知受信DTO"),
                    debugInfo);

        }

        //4. 戻り値としてLSCバージョン通知受信DTO[versionNoticeRcvDto]を返却し、処理を終了する。
        return versionNoticeRcvDto;

    }

    /**
     * オブジェクト変換(LAN信号変換装置系切替要求)メソッド
     * 
     * LAN信号変換装置から受信したデータをLAN信号変換装置系切替要求に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscSwitchNoticeRcvDto LSC切替通知受信DTO
     */
    private LscSwitchNoticeRcvDto convLscSwitchToObject(byte[] rcvData, LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LAN信号変換装置LSC切替通知受信DTO[lscSwitchNoticeRcvDto]を生成する。
        LscSwitchNoticeRcvDto lscSwitchNoticeRcvDto = new LscSwitchNoticeRcvDto();

        //2. LAN信号変換装置LSC切替通知受信DTO[lscSwitchNoticeRcvDto]に以下の値を設定する。
        lscSwitchNoticeRcvDto.comHeaderDto = comHeaderDto;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LAN信号変換装置系切替要求(LSC切替通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscSwitchNoticeRcvDto", lscSwitchNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //3. データ検証を行う。

        //3.1. LAN信号変換装置LSC切替通知受信DTO[lscSwitchNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSwitchNoticeRcvDto>> checkResult = this.validateData(lscSwitchNoticeRcvDto);

        //3.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //3.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //3.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSwitchNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LAN信号変換装置系切替要求", "LSC切替通知受信DTO"),
                    debugInfo);

        }

        //4. 戻り値としてLAN信号変換装置LSC切替通知受信DTO[lscSwitchNoticeRcvDto]を返却し、処理を終了する。
        return lscSwitchNoticeRcvDto;

    }

    /**
     * オブジェクト変換(保守通話状態通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを保守通話状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscMaintenanceCallStatusNoticeRcvDto LSC保守通話状態通知受信DTO
     */
    private LscMaintenanceCallStatusNoticeRcvDto convMaintenanceCallStatusToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]を生成する。
        LscMaintenanceCallStatusNoticeRcvDto maintenanceCallStatusNoticeRcvDto =
                new LscMaintenanceCallStatusNoticeRcvDto();

        //2. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に以下の値を設定する。
        maintenanceCallStatusNoticeRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に以下の値を設定する。
            maintenanceCallStatusNoticeRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            maintenanceCallStatusNoticeRcvDto.baseStaEquipNo = Arrays.copyOfRange(rcvData, 19, 21);
            maintenanceCallStatusNoticeRcvDto.maintenanceCallCtrl =
                    new String(Arrays.copyOfRange(rcvData, 21, 22), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 保守通話状態通知(LSC保守通話状態通知受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("maintenanceCallStatusNoticeRcvDto", maintenanceCallStatusNoticeRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscMaintenanceCallStatusNoticeRcvDto>> checkResult =
                this.validateData(maintenanceCallStatusNoticeRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscMaintenanceCallStatusNoticeRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "保守通話状態通知", "LSC保守通話状態通知受信DTO"),
                    debugInfo);

        }

        //5. 戻り値としてLSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]を返却し、処理を終了する。
        return maintenanceCallStatusNoticeRcvDto;

    }

    /**
     * オブジェクト変換(基地局選択状態通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを基地局選択状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @return LscBaseStaSelectStatusResRcvDto LSC基地局選択状態応答受信DTO
     */
    private LscBaseStaSelectStatusResRcvDto convBaseStaSelectStatusNoticeToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]を生成する。
        LscBaseStaSelectStatusResRcvDto baseStaSelectStatusResRcvDto = new LscBaseStaSelectStatusResRcvDto();

        //2. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]に以下の値を設定する。
        baseStaSelectStatusResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]に以下の値を設定する。
            baseStaSelectStatusResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            baseStaSelectStatusResRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            baseStaSelectStatusResRcvDto.order = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 基地局選択状態通知(LSC基地局選択状態応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("baseStaSelectStatusResRcvDto", baseStaSelectStatusResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscBaseStaSelectStatusResRcvDto>> checkResult =
                this.validateData(baseStaSelectStatusResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscBaseStaSelectStatusResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "基地局選択状態通知", "LSC基地局選択状態応答受信DTO"),
                    debugInfo);

        }

        //5. 戻り値として基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]を返却し、処理を終了する。
        return baseStaSelectStatusResRcvDto;

    }

    /**
     * オブジェクト変換(基地局選択状態通知応答)メソッド
     * 
     * LAN信号変換装置から受信したデータを基地局選択状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscBaseStaSelectStatusResRcvDto LSC基地局選択状態応答受信DTO
     */
    private LscBaseStaSelectStatusResRcvDto convBaseStaSelectStatusNoticeResToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto, MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]を生成する。
        LscBaseStaSelectStatusResRcvDto baseStaSelectStatusResRcvDto = new LscBaseStaSelectStatusResRcvDto();

        //2. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]に以下の値を設定する。
        baseStaSelectStatusResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. 基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]に以下の値を設定する。
            baseStaSelectStatusResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            baseStaSelectStatusResRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            baseStaSelectStatusResRcvDto.order = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 基地局選択状態通知応答(LSC基地局選択状態応答受信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("baseStaSelectStatusResRcvDto", baseStaSelectStatusResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscBaseStaSelectStatusResRcvDto>> checkResult =
                this.validateData(baseStaSelectStatusResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscBaseStaSelectStatusResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "基地局選択状態通知", "LSC基地局選択状態応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値として基地局選択状態応答受信DTO[baseStaSelectStatusResRcvDto]を返却し、処理を終了する。
        return baseStaSelectStatusResRcvDto;

    }

    /**
     * オブジェクト変換(規制制御状態通知応答)メソッド
     * 
     * LAN信号変換装置から受信したデータを規制制御状態通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscReglationCtrlStatusResRcvDto LSC規制制御状態応答受信DTO
     */
    private LscReglationCtrlStatusResRcvDto convReglationCtrlStatusResToObject(byte[] rcvData,
            LscComHeaderDto comHeaderDto, MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]を生成する。
        LscReglationCtrlStatusResRcvDto reglationCtrlStatusResRcvDto = new LscReglationCtrlStatusResRcvDto();

        //2. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に以下の値を設定する。
        reglationCtrlStatusResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に以下の値を設定する。
            reglationCtrlStatusResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            reglationCtrlStatusResRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        //4. SACCH/FACCH情報を設定する。

        //4.1. SACCH/FACCH情報[reglationCtrlStatusResRcvDto.ctrlChInfo]を生成する。
        reglationCtrlStatusResRcvDto.ctrlChInfo = new LscReglationCtrlStatusResRcvDto.CtrlChInfo();

        try {

            //4.2. SACCH/FACCH情報[reglationCtrlStatusResRcvDto.ctrlChInfo]に以下の値を設定する。
            reglationCtrlStatusResRcvDto.ctrlChInfo.messageType = Byte.toUnsignedInt(rcvData[22]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.reglationInfo = Byte.toUnsignedInt(rcvData[23]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.dispatchCmdDetail = Byte.toUnsignedInt(rcvData[24]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.commReglationDetail = Byte.toUnsignedInt(rcvData[25]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.emergSignalDetail = Byte.toUnsignedInt(rcvData[26]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.forcedDisconnectDetail = Byte.toUnsignedInt(rcvData[27]);
            reglationCtrlStatusResRcvDto.ctrlChInfo.userData =
                    new String(
                            Arrays.copyOfRange(rcvData, 28,
                                    Integer.parseInt(reglationCtrlStatusResRcvDto.comHeaderDto.telegramLength)),
                            "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 規制制御状態通知応答(LSC規制制御状態応答受信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("reglationCtrlStatusResRcvDto", reglationCtrlStatusResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //5. データ検証を行う。

        //5.1. LSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscReglationCtrlStatusResRcvDto>> checkResult =
                this.validateData(reglationCtrlStatusResRcvDto);

        //5.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //5.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //5.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscReglationCtrlStatusResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "規制制御状態通知", "LSC規制制御状態応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //6. 戻り値としてLSC規制制御状態応答受信DTO[reglationCtrlStatusResRcvDto]を返却し、処理を終了する。
        return reglationCtrlStatusResRcvDto;

    }

    /**
     * オブジェクト変換(セレコール応答受信通知)メソッド
     * 
     * LAN信号変換装置から受信したデータをセレコール応答受信通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscSelcallResRcvDto LSCセレコール応答受信DTO
     */
    private LscSelcallResRcvDto convSelcallResRcvNoticeToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. セレコール応答受信DTO[selcallResRcvDto]を生成する。
        LscSelcallResRcvDto selcallResRcvDto = new LscSelcallResRcvDto();

        //2. セレコール応答受信DTO[selcallResRcvDto]に以下の値を設定する。
        selcallResRcvDto.comHeaderDto = comHeaderDto;

        try {
            //3. セレコール応答受信DTO[selcallResRcvDto]に以下の値を設定する。
            selcallResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            selcallResRcvDto.receptionRslt = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - セレコール応答受信通知(LSCセレコール応答受信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("selcallResRcvDto", selcallResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSelcallResRcvDto>> checkResult =
                this.validateData(selcallResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSelcallResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "セレコール応答受信通知", "LSCセレコール応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値としてセレコール応答受信DTO[selcallResRcvDto]を返却し、処理を終了する。
        return selcallResRcvDto;

    }

    /**
     * オブジェクト変換(通信設定応答通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを通信設定応答通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO  
     * @param messageResultDto 電文実行結果DTO
     * @return LscCommConfResRcvDto LSC通信設定応答受信DTO
     */
    private LscCommConfResRcvDto convCommConfResNoticeToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC通信設定応答受信DTO[lscCommConfResRcvDto]を生成する。
        LscCommConfResRcvDto lscCommConfResRcvDto = new LscCommConfResRcvDto();

        //2. LSC通信設定応答受信DTO[lscCommConfResRcvDto]に以下の値を設定する。
        lscCommConfResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC通信設定応答受信DTO[lscCommConfResRcvDto]に以下の値を設定する。
            lscCommConfResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            lscCommConfResRcvDto.baseStaNo = new String(Arrays.copyOfRange(rcvData, 19, 22), "MS932");
            lscCommConfResRcvDto.commType = new String(Arrays.copyOfRange(rcvData, 22, 23), "MS932");
            lscCommConfResRcvDto.conf = new String(Arrays.copyOfRange(rcvData, 23, 24), "MS932");
            lscCommConfResRcvDto.rslt = new String(Arrays.copyOfRange(rcvData, 24, 25), "MS932");
            lscCommConfResRcvDto.incallDestNoLength = new String(Arrays.copyOfRange(rcvData, 25, 26), "MS932");
            lscCommConfResRcvDto.incallDestNo = new String(Arrays.copyOfRange(rcvData, 26, 34), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 通信設定応答通知(LSC通信設定応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommConfResRcvDto", lscCommConfResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommConfResRcvDto>> checkResult =
                this.validateData(lscCommConfResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommConfResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "通信設定応答通知", "LSC通信設定応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値としてLSC通信設定応答受信DTO[lscCommConfResRcvDto]を返却し、処理を終了する。
        return lscCommConfResRcvDto;

    }

    /**
     * オブジェクト変換(通信開始応答通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを通信開始応答通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscCommStartResRcvDto LSC通信開始応答受信DTO
     */
    private LscCommStartResRcvDto convCommStartResNoticeToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC通信開始応答受信DTO[lscCommStartResRcvDto]を生成する。
        LscCommStartResRcvDto lscCommStartResRcvDto = new LscCommStartResRcvDto();

        //2. LSC通信開始応答受信DTO[lscCommStartResRcvDto]に以下の値を設定する。
        lscCommStartResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC通信開始応答受信DTO[lscCommStartResRcvDto]に以下の値を設定する。
            lscCommStartResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            lscCommStartResRcvDto.receptionRslt = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 通信開始応答通知(LSC通信開始応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommStartResRcvDto", lscCommStartResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommStartResRcvDto>> checkResult =
                this.validateData(lscCommStartResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommStartResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "通信開始応答通知", "LSC通信開始応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値としてLSC通信開始応答受信DTO[lscCommStartResRcvDto]を返却し、処理を終了する。
        return lscCommStartResRcvDto;

    }

    /**
     * オブジェクト変換(通信終了応答通知)メソッド
     * 
     * LAN信号変換装置から受信したデータを通信終了応答通知に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscCommEndResRcvDto LSC通信終了応答受信DTO
     */
    private LscCommEndResRcvDto convCommEndResNoticeToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. LSC通信終了応答受信DTO[lscCommEndResRcvDto]を生成する。
        LscCommEndResRcvDto lscCommEndResRcvDto = new LscCommEndResRcvDto();

        //2. LSC通信終了応答受信DTO[lscCommEndResRcvDto]に以下の値を設定する。
        lscCommEndResRcvDto.comHeaderDto = comHeaderDto;

        try {

            //3. LSC通信終了応答受信DTO[lscCommEndResRcvDto]に以下の値を設定する。
            lscCommEndResRcvDto.chCode = new String(Arrays.copyOfRange(rcvData, 16, 19), "MS932");
            lscCommEndResRcvDto.receptionRslt = new String(Arrays.copyOfRange(rcvData, 19, 20), "MS932");

        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 通信終了応答通知(LSC通信終了応答受信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommEndResRcvDto", lscCommEndResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommEndResRcvDto>> checkResult =
                this.validateData(lscCommEndResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommEndResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "通信終了応答通知", "LSC通信終了応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値としてLSC通信終了応答受信DTO[lscCommEndResRcvDto]を返却し、処理を終了する。
        return lscCommEndResRcvDto;

    }

    /**
     * オブジェクト変換(時刻設定変更応答)メソッド
     * 
     * LAN信号変換装置から受信したデータを時刻設定変更応答に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscTimeConfChangeResRcvDto LSC時間設定変更応答受信DTO
     */
    private LscTimeConfChangeResRcvDto convTimeConfChangeResToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. 時間設定変更応答受信DTO[timeConfChangeResRcvDto]を生成する。
        LscTimeConfChangeResRcvDto timeConfChangeResRcvDto = new LscTimeConfChangeResRcvDto();

        //2. 時間設定変更応答受信DTO[timeConfChangeResRcvDto]に以下の値を設定する。
        timeConfChangeResRcvDto.comHeaderDto = comHeaderDto;

        //3. 時間設定変更応答受信DTO[timeConfChangeResRcvDto]に以下の値を設定する。
        try {
            timeConfChangeResRcvDto.rslt = new String(Arrays.copyOfRange(rcvData, 16, 17), "MS932");
        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 時刻設定変更応答(LSC時間設定変更応答受信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("timeConfChangeResRcvDto", timeConfChangeResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscTimeConfChangeResRcvDto>> checkResult =
                this.validateData(timeConfChangeResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscTimeConfChangeResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "時刻設定変更応答", "LSC時間設定変更応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値として時間設定変更応答受信DTO[timeConfChangeResRcvDto]を返却し、処理を終了する。
        return timeConfChangeResRcvDto;

    }

    /**
     * オブジェクト変換(チャネル占有通信設定応答)メソッド
     * 
     * LAN信号変換装置から受信したデータをチャネル占有通信設定応答に変換する。
     * @param rcvData 受信データ
     * @param comHeaderDto LSC共通ヘッダDTO
     * @param messageResultDto 電文実行結果DTO
     * @return LscChOccCommConfResRcvDto LSCチャネル占有通信設定応答受信DTO
     */
    private LscChOccCommConfResRcvDto convChOccCommConfResToObject(byte[] rcvData, LscComHeaderDto comHeaderDto,
            MessageResultDto messageResultDto) {

        // 【初期化処理】
        // クラス情報
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        //1. チャネル占有通信設定応答受信DTO[chOccCommConfResRcvDto]を生成する。
        LscChOccCommConfResRcvDto chOccCommConfResRcvDto = new LscChOccCommConfResRcvDto();

        //2. チャネル占有通信設定応答受信DTO[chOccCommConfResRcvDto]に以下の値を設定する。
        chOccCommConfResRcvDto.comHeaderDto = comHeaderDto;

        //3. チャネル占有通信設定応答受信DTO[chOccCommConfResRcvDto]に以下の値を設定する。
        try {
            chOccCommConfResRcvDto.rslt = new String(Arrays.copyOfRange(rcvData, 16, 17), "MS932");
        } catch (UnsupportedEncodingException e) {
            // TODO 自動生成された catch ブロック
            e.printStackTrace();
        }

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【受信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - チャネル占有通信設定応答(LSCチャネル占有通信設定応答受信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("chOccCommConfResRcvDto", chOccCommConfResRcvDto);

        logger.debug(callerClazz, "", debugInfo);

        //4. データ検証を行う。

        //4.1. LSC保守通話状態通知受信DTO[maintenanceCallStatusNoticeRcvDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscChOccCommConfResRcvDto>> checkResult =
                this.validateData(chOccCommConfResRcvDto);

        //4.2. チェック結果[checkResult]のサイズにより以下を実行する。

        //4.2.1. チェック結果[checkResult]のサイズが0件以外の場合、以下を実行する。
        if (checkResult.size() != 0) {

            //4.2.1.1. エラーログを出力する。
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscChOccCommConfResRcvDto> checkItem : checkResult) {

                debugInfo.put(checkItem.getPropertyPath() + "=" + checkItem.getInvalidValue(), checkItem.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00121,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "チャネル占有通信設定応答",
                            "LSCチャネル占有通信設定応答受信DTO"),
                    debugInfo);

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.RECEIVE_DATA_CHECK_ERROR;

            // 上記以外の場合、以下を実行する。
        } else {

            // 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;
        }

        //5. 戻り値としてチャネル占有通信設定応答受信DTO[chOccCommConfResRcvDto]を返却し、処理を終了する。
        return chOccCommConfResRcvDto;

    }
}
