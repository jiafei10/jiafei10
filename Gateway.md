 * LscGateway.java.
 *
 * All rights reserved, Copyright(C) 2022 Oki Electric Industry Co.,Ltd.
 */

package com.oki.ofits.common.api.extif.lsc;

import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

import javax.validation.ConstraintViolation;

import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;

import com.oki.ofits.common.api.extif.TcpGateway;
import com.oki.ofits.common.api.extif.lsc.LscGatewayConfiguration.ISendTcpData;
import com.oki.ofits.common.constant.Const;
import com.oki.ofits.common.constant.SystemCode;
import com.oki.ofits.common.controlparam.SystemControlParam.IfLscControlParam;
import com.oki.ofits.common.logging.AppLogger;
import com.oki.ofits.common.message.MsgId;
import com.oki.ofits.logic.model.extif.MessageResultDto;
import com.oki.ofits.logic.model.extif.lsc.LscBaseStaSelectStatusReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscBaseStaSelectStatusResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscChOccCommConfReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscChOccCommConfResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommConfReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommConfResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommEndReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommEndResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommStartReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscCommStartResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscMobileStaAreaInfoGetResSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscOpeStartResSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscReglationCtrlStatusReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscReglationCtrlStatusResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscSelcallReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscSelcallResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscStatusReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscSwitchResSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscTimeConfChangeReqSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscTimeConfChangeResRcvDto;
import com.oki.ofits.logic.model.extif.lsc.LscTimeResSndDto;
import com.oki.ofits.logic.model.extif.lsc.LscVersionResSndDto;

/**
 * LAN信号変換装置Gatewayクラス.
 * <p>
 * LAN信号変換装置に対するIFの実装を提供する。
 *
 *  ① 受信電文を解析し対応するControllerに通知する。
 *  ② 無線回線制御装置に対し時刻を応答する。
 *  ③ 指令台より無線通信を行う基地局を個別選択する。
 *  ④ 無線回線制御装置の状態を要求する。
 *  ⑤ 無線回線制御装置に対し規制制御を要求する。
 *  ⑥ 無線回線制御装置からの無線システムの運用開始通知を受け、運用開始要求結果を応答する。
 *  ⑦ 無線回線制御装置に対し通信設定を要求する。
 *  ⑧ 無線回線制御装置に対し通信開始を要求する。
 *  ⑨ 無線回線制御装置に対し通信終了を要求する。
 *  ⑩ 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
 *  ⑪ 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
 *  ⑫ 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
 *  ⑬ 無線回線制御装置からのバージョン情報の通知要求を受け、バージョン情報を応答する。
 *  ⑭ 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
 *  ⑮ 無線回線制御装置に対しシステムの時間設定変更を要求する。
 * </p>
 */
@Component
public class LscGateway extends TcpGateway implements ILscGateway {

    /** コンテキスト. */
    private ApplicationContext context;

    /** アプリケーションロガー. */
    private AppLogger logger;

    /** LAN信号変換装置要求応答レシーバー. */
    private LscReqResReceiver lscReqResReceiver;

    @Autowired
    public LscGateway(ApplicationContext context, AppLogger logger, LscReqResReceiver lscReqResReceiver) {
        this.context = context;
        this.logger = logger;
        this.lscReqResReceiver = lscReqResReceiver;
    }

    /**
     * 時間応答送信メソッド
     * 
     * 無線回線制御装置に対し時刻を応答する。
     * @param lscTimeResSndDto LSC時間応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndTimeRes(LscTimeResSndDto lscTimeResSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC時間応答送信DTO.LSC共通ヘッダDTO[lscTimeResSndDto.comHeaderDto]に以下の値を設定する。
        lscTimeResSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscTimeResSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_TIME_NOTICE;
        lscTimeResSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_TIME_NOTICE;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 時間応答送信(LSC時間応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscTimeResSndDto", lscTimeResSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC時間応答送信DTO[lscTimeResSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscTimeResSndDto>> checkSendDataResult = this.validateData(lscTimeResSndDto);

        //1.3. 電文実行結果DTO[melscageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscTimeResSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "時間応答送信", "LSC時間応答送信DTO"), debugInfo);

            //1.4.1.1. 電文実行結果DTO[melscageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[melscageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscTimeResSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 時間応答送信(LSC時間応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {
            sendTcpData.sendTcpData(sndData);

            //2.2.2.1. 電文実行結果DTO[melscageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[melscageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.CONNECT_ERROR;

            //3. 戻り値として電文実行結果DTO[melscageResultDto]を返却し、処理を終了する。
        }
        return messageResultDto;

    }

    /**
     * 基地局選択状態要求送信メソッド
     * 
     * 指令台より無線通信を行う基地局を個別選択する。
     * @param lscBaseStaSelectStatusReqSndDto LSC基地局選択状態要求送信DTO
     * @param lscBaseStaSelectStatusResRcvDto LSC基地局選択状態応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndBaseStaSelectStatusReq(
            LscBaseStaSelectStatusReqSndDto lscBaseStaSelectStatusReqSndDto,
            LscBaseStaSelectStatusResRcvDto lscBaseStaSelectStatusResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC基地局選択状態要求送信DTO.LSC共通ヘッダDTO[baseStaSelectLscStatusReqSndDto.comHeaderDto]に以下の値を設定する。
        lscBaseStaSelectStatusReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscBaseStaSelectStatusReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_BASE_STA_SELECT_REQ;
        lscBaseStaSelectStatusReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_BASE_STA_SELECT_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 基地局選択状態要求送信(LSC基地局選択状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscBaseStaSelectStatusReqSndDto", lscBaseStaSelectStatusReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC基地局選択状態要求送信DTO[baseStaSelectLscStatusReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscBaseStaSelectStatusReqSndDto>> checkSendDataResult =
                this.validateData(lscBaseStaSelectStatusReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscBaseStaSelectStatusReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "基地局選択状態要求送信", "LSC基地局選択状態要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        }

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscBaseStaSelectStatusReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 基地局選択状態要求送信(LSC基地局選択状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.CONNECT_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscBaseStaSelectStatusReqSndDto.comHeaderDto,
                        lscBaseStaSelectStatusReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscBaseStaSelectStatusResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * LSC状態要求送信メソッド
     * 
     * 無線回線制御装置の状態を要求する。
     * @param lscStatusReqSndDto LSC状態要求送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscStatusReq(LscStatusReqSndDto lscStatusReqSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC状態要求送信DTO.LSC共通ヘッダDTO[lscStatusReqSndDto.comHeaderDto]に以下の値を設定する。
        lscStatusReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscStatusReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_STATUS_REQ;
        lscStatusReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_STATUS_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LSC状態要求送信(LSC状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscStatusReqSndDto", lscStatusReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC状態要求送信DTO[lscStatusReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscStatusReqSndDto>> checkSendDataResult = this.validateData(lscStatusReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscStatusReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LSC状態要求送信", "LSC状態要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscStatusReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - LSC状態要求送信(LSC状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2.2.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

        }
        //3. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
        return messageResultDto;

    }

    /**
     * 規制制御状態要求送信メソッド
     * 
     * 無線回線制御装置に対し規制制御を要求する。
     * @param lscReglationCtrlStatusReqSndDto LSC規制制御状態要求送信DTO
     * @param lscReglationCtrlStatusResRcvDto LSC規制制御状態応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndReglationCtrlStatusReq(
            LscReglationCtrlStatusReqSndDto lscReglationCtrlStatusReqSndDto,
            LscReglationCtrlStatusResRcvDto lscReglationCtrlStatusResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC規制制御状態要求送信DTO.LSC共通ヘッダDTO[reglationCtrlLscStatusReqSndDto.comHeaderDto]に以下の値を設定する。
        lscReglationCtrlStatusReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscReglationCtrlStatusReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_REGULATION_REQ;
        lscReglationCtrlStatusReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_REGULATION_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 規制制御状態要求送信(LSC規制制御状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscReglationCtrlStatusReqSndDto", lscReglationCtrlStatusReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC規制制御状態要求送信DTO[reglationCtrlLscStatusReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscReglationCtrlStatusReqSndDto>> checkSendDataResult =
                this.validateData(lscReglationCtrlStatusReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscReglationCtrlStatusReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "規制制御状態要求送信", "LSC規制制御状態要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        }

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscReglationCtrlStatusReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 規制制御状態要求送信(LSC規制制御状態要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscReglationCtrlStatusReqSndDto.comHeaderDto,
                        lscReglationCtrlStatusReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscReglationCtrlStatusResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * 運用開始応答送信メソッド
     * 
     * 無線回線制御装置からの無線システムの運用開始通知を受け、運用開始要求結果を応答する。
     * @param lscOpeStartResSndDto LSC運用開始応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndOpeStartRes(LscOpeStartResSndDto lscOpeStartResSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC運用開始応答送信DTO.LSC共通ヘッダDTO[opeStartResSndDto.comHeaderDto]に以下の値を設定する。
        lscOpeStartResSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscOpeStartResSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_OPE_START_RES_NOTICE;
        lscOpeStartResSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_OPE_START_RES_NOTICE;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 運用開始応答送信(LSC運用開始応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscOpeStartResSndDto", lscOpeStartResSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC運用開始応答送信DTO[opeStartResSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscOpeStartResSndDto>> checkSendDataResult = this.validateData(lscOpeStartResSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscOpeStartResSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "運用開始応答送信", "LSC運用開始応答送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscOpeStartResSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 運用開始応答送信(LSC運用開始応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {
            sendTcpData.sendTcpData(sndData);

            //2.2.2.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.2. 上記以外の場合、何もしない。
        }

        //3. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
        return messageResultDto;

    }

    /**
     * LSC通信設定要求送信メソッド
     * 
     * 無線回線制御装置に対し通信設定を要求する。
     * @param lscCommConfReqSndDto LSC通信設定要求送信DTO
     * @param lscCommConfResRcvDto LSC通信設定応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommConfReq(LscCommConfReqSndDto lscCommConfReqSndDto,
            LscCommConfResRcvDto lscCommConfResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC通信設定要求送信DTO[lscCommConfReqSndDto]に以下の値を設定する。
        lscCommConfReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscCommConfReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_TELECOM_CONF_REQ;
        lscCommConfReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_TELECOM_CONF_REQ;
        lscCommConfReqSndDto.baseStaNo = Const.IF_LSC_BASE_STA_NO_NONE_SELECT;
        lscCommConfReqSndDto.reserve = Const.IF_LSC_SPARE;
        lscCommConfReqSndDto.incallDestNoLength = Const.IF_LSC_INCALL_DEST_NO_LEN;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LSC通信設定要求送信(LSC通信設定要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommConfReqSndDto", lscCommConfReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC通信設定要求送信DTO[lscCommConfReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommConfReqSndDto>> checkSendDataResult = this.validateData(lscCommConfReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommConfReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LSC通信設定要求送信", "LSC通信設定要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscCommConfReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - LSC通信設定要求送信(LSC通信設定要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {
            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscCommConfReqSndDto.comHeaderDto, lscCommConfReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscCommConfResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * LSC通信開始要求送信メソッド
     * 
     * 無線回線制御装置に対し通信開始を要求する。
     * @param lscCommStartReqSndDto LSC通信開始要求送信DTO
     * @param lscCommStartResRcvDto LSC通信開始応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommStartReq(LscCommStartReqSndDto lscCommStartReqSndDto,
            LscCommStartResRcvDto lscCommStartResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC通信開始要求送信DTO.LSC共通ヘッダDTO[lscCommStartReqSndDto.comHeaderDto]に以下の値を設定する。
        lscCommStartReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscCommStartReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_TELECOM_START_REQ;
        lscCommStartReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_TELECOM_START_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LSC通信開始要求送信(LSC通信開始要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommStartReqSndDto", lscCommStartReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC通信開始要求送信DTO[lscCommStartReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommStartReqSndDto>> checkSendDataResult = this.validateData(lscCommStartReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommStartReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LSC通信開始要求送信", "LSC通信開始要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscCommStartReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - LSC通信開始要求送信(LSC通信開始要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscCommStartReqSndDto.comHeaderDto, lscCommStartReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscCommStartResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * LSC通信終了要求送信メソッド
     * 
     * 無線回線制御装置に対し通信終了を要求する。
     * @param lscCommEndReqSndDto LSC通信終了要求送信DTO
     * @param lscCommEndResRcvDto LSC通信終了応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommEndReq(LscCommEndReqSndDto lscCommEndReqSndDto,
            LscCommEndResRcvDto lscCommEndResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC通信終了要求送信DTO.LSC共通ヘッダDTO[lscCommEndReqSndDto.comHeaderDto]に以下の値を設定する。
        lscCommEndReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscCommEndReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_TELECOM_END_REQ;
        lscCommEndReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_TELECOM_END_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LSC通信終了要求送信(LSC通信終了要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscCommEndReqSndDto", lscCommEndReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC通信終了要求送信DTO[lscCommEndReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscCommEndReqSndDto>> checkSendDataResult = this.validateData(lscCommEndReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscCommEndReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LSC通信終了要求送信", "LSC通信終了要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscCommEndReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - LSC通信終了要求送信(LSC通信終了要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscCommEndReqSndDto.comHeaderDto, lscCommEndReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscCommEndResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * セレコール要求送信メソッド
     * 
     * 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
     * @param lscSelcallReqSndDto LSCセレコール要求送信DTO
     * @param lscSelcallResRcvDto LSCセレコール応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndSelcallReq(LscSelcallReqSndDto lscSelcallReqSndDto,
            LscSelcallResRcvDto lscSelcallResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSCセレコール要求送信DTO[selcallReqSndDto]に以下の値を設定する。
        lscSelcallReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscSelcallReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_SELECALL_RES_NOTICE;
        lscSelcallReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_SELECALL_RES_NOTICE;
        lscSelcallReqSndDto.commType = Const.IF_LSC_TELECOM_TYPE_SELECALL_RES_NOTICE;
        lscSelcallReqSndDto.outcallSrcNoLength = Const.IF_LSC_OUTCALL_SRC_NO_LEN;
        lscSelcallReqSndDto.incallDestNoLength = Const.IF_LSC_INCALL_DEST_NO_LEN;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - セレコール要求送信(LSCセレコール要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscSelcallReqSndDto", lscSelcallReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSCセレコール要求送信DTO[selcallReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSelcallReqSndDto>> checkSendDataResult = this.validateData(lscSelcallReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSelcallReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "セレコール要求送信", "LSCセレコール要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscSelcallReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - セレコール要求送信(LSCセレコール要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscSelcallReqSndDto.comHeaderDto, lscSelcallReqSndDto.chCode);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscSelcallResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * チャネル占有通信設定要求送信メソッド
     * 
     * 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
     * @param lscChOccCommConfReqSndDto LSCチャネル占有通信設定要求送信DTO
     * @param lscChOccCommConfResRcvDto LSCチャネル占有通信設定応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndChOccCommConfReq(LscChOccCommConfReqSndDto lscChOccCommConfReqSndDto,
            LscChOccCommConfResRcvDto lscChOccCommConfResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSCチャネル占有通信設定要求送信DTO.LSC共通ヘッダDTO[chOccLscCommConfReqSndDto.comHeaderDto]に以下の値を設定する。
        lscChOccCommConfReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscChOccCommConfReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_CHANNEL_TELECOM_CONF;
        lscChOccCommConfReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_CHANNEL_TELECOM_CONF;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - チャネル占有通信設定要求送信(LSCチャネル占有通信設定要求送信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscChOccCommConfReqSndDto", lscChOccCommConfReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSCチャネル占有通信設定要求送信DTO[chOccLscCommConfReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscChOccCommConfReqSndDto>> checkSendDataResult =
                this.validateData(lscChOccCommConfReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscChOccCommConfReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "チャネル占有通信設定要求送信",
                            "LSCチャネル占有通信設定要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscChOccCommConfReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - チャネル占有通信設定要求送信(LSCチャネル占有通信設定要求送信DTO)",
                        IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscChOccCommConfReqSndDto.comHeaderDto, null);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscChOccCommConfResRcvDto);
        }

        return messageResultDto;

    }

    /**
     * 移動局在圏情報取得応答送信メソッド
     * 
     * 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
     * @param lscMobileStaAreaInfoGetResSndDto LSC移動局在圏情報取得応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndMobileStaAreaInfoGetRes(
            LscMobileStaAreaInfoGetResSndDto lscMobileStaAreaInfoGetResSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC移動局在圏情報取得応答送信DTO.LSC共通ヘッダDTO[mobileStaAreaInfoGetResSndDto.comHeaderDto]に以下の値を設定する。
        lscMobileStaAreaInfoGetResSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscMobileStaAreaInfoGetResSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_MOBILE_STA_AREA_RES;
        lscMobileStaAreaInfoGetResSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_MOBILE_STA_AREA_RES;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 移動局在圏情報取得応答送信(LSC移動局在圏情報取得応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscMobileStaAreaInfoGetResSndDto", lscMobileStaAreaInfoGetResSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC移動局在圏情報取得応答送信DTO[mobileStaAreaInfoGetResSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscMobileStaAreaInfoGetResSndDto>> checkSendDataResult =
                this.validateData(lscMobileStaAreaInfoGetResSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscMobileStaAreaInfoGetResSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "移動局在圏情報取得応答送信",
                            "LSC移動局在圏情報取得応答送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscMobileStaAreaInfoGetResSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 移動局在圏情報取得応答送信(LSC移動局在圏情報取得応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        //2.3. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
        messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

        //3. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
        return messageResultDto;

    }

    /**
     * バージョン応答送信メソッド
     * 
     * 無線回線制御装置からのバージョン情報の通知要求を受け、バージョン情報を応答する。
     * @param lscVersionResSndDto LSCバージョン応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndVersionRes(LscVersionResSndDto lscVersionResSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSCバージョン応答送信DTO.LSC共通ヘッダDTO[versionResSndDto.comHeaderDto]に以下の値を設定する。
        lscVersionResSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscVersionResSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_VER_NOTICE;
        lscVersionResSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_VER_NOTICE;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - バージョン応答送信(LSCバージョン応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscVersionResSndDto", lscVersionResSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSCバージョン応答送信DTO[versionResSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscVersionResSndDto>> checkSendDataResult = this.validateData(lscVersionResSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscVersionResSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "バージョン応答送信", "LSCバージョン応答送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        } else {}

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscVersionResSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - バージョン応答送信(LSCバージョン応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {
            sendTcpData.sendTcpData(sndData);

            //2.2.2.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.2. 上記以外の場合、以下を実行する。
        }

        //3. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
        return messageResultDto;

    }

    /**
     * LAN信号変換装置切替応答送信メソッド
     * 
     * 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
     * @param lscSwitchResSndDto LSC切替応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscSwitchRes(LscSwitchResSndDto lscSwitchResSndDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LAN信号変換装置LSC切替応答送信DTO.LSC共通ヘッダDTO[lscSwitchResSndDto.comHeaderDto]に以下の値を設定する。
        lscSwitchResSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscSwitchResSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_LAN_SWITCH_RES;
        lscSwitchResSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_LAN_SWITCH_RES;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - LAN信号変換装置切替応答送信(LSC切替応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscSwitchResSndDto", lscSwitchResSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LAN信号変換装置LSC切替応答送信DTO[lscSwitchResSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscSwitchResSndDto>> checkSendDataResult = this.validateData(lscSwitchResSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscSwitchResSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "LAN信号変換装置切替応答送信", "LSC切替応答送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        }

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscSwitchResSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - LAN信号変換装置切替応答送信(LSC切替応答送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {
            sendTcpData.sendTcpData(sndData);

            //2.2.2.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.NORMAL;

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.2. 上記以外の場合、以下を実行する。
        }

        //3. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
        return messageResultDto;

    }

    /**
     * 時間設定変更要求送信メソッド
     * 
     * 無線回線制御装置に対しシステムの時間設定変更を要求する。
     * @param lscTimeConfChangeReqSndDto LSC時間設定変更要求送信DTO
     * @param lscTimeConfChangeResRcvDto LSC時間設定変更応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndTimeConfChangeReq(LscTimeConfChangeReqSndDto lscTimeConfChangeReqSndDto,
            LscTimeConfChangeResRcvDto lscTimeConfChangeResRcvDto) {

        //1. 送信電文を作成する。

        //1.1. P01：LSC時間設定変更要求送信DTO.LSC共通ヘッダDTO[timeConfChangeReqSndDto.comHeaderDto]に以下の値を設定する。
        lscTimeConfChangeReqSndDto.comHeaderDto.telegramSeparator = Const.IF_LSC_TELE_SEPARATOR;
        lscTimeConfChangeReqSndDto.comHeaderDto.telegramLength = Const.IF_LSC_TELE_LEN_TIME_CONF_CHANGE_REQ;
        lscTimeConfChangeReqSndDto.comHeaderDto.telegramIdent = Const.IF_LSC_TELE_ID_TIME_CONF_CHANGE_REQ;

        // クラス情報を取得
        Class<?> callerClazz = new Object() {}.getClass();
        Map<String, Object> debugInfo = null;

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文DTO出力】",
                String.format("LAN信号変換装置(%s) - 時間設定変更要求送信(LSC時間設定変更要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("lscTimeConfChangeReqSndDto", lscTimeConfChangeReqSndDto);

        logger.debug(callerClazz, "", debugInfo);

        //1.2. P01：LSC時間設定変更要求送信DTO[timeConfChangeReqSndDto]に対してバリデーションチェックを実行する。
        Set<ConstraintViolation<LscTimeConfChangeReqSndDto>> checkSendDataResult =
                this.validateData(lscTimeConfChangeReqSndDto);

        //1.3. 電文実行結果DTO[messageResultDto]を生成する。
        MessageResultDto messageResultDto = new MessageResultDto();

        //1.4. 送信データチェック結果[checkSendDataResult]のサイズにより以下を実行する。

        //1.4.1. 送信データチェック結果[checkSendDataResult]のサイズが0件以外の場合、以下を実行する。
        if (checkSendDataResult.size() != 0) {

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            for (ConstraintViolation<LscTimeConfChangeReqSndDto> item : checkSendDataResult) {

                debugInfo.put(item.getPropertyPath() + "=" + item.getInvalidValue(), item.getMessage());

            }

            logger.error(callerClazz, MsgId.EAE_03_00120,
                    Arrays.asList("LAN信号変換装置", IfLscControlParam.SEND_IPADDRESS, "時間設定変更要求送信", "LSC時間設定変更要求送信DTO"),
                    debugInfo);

            //1.4.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_DATA_CHECK_ERROR;

            //1.4.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //1.4.2. 上記以外の場合、何もしない。
        }

        //2. 電文送信を呼び出し、項番1で作成した電文を送信する。
        byte[] sndData = lscTimeConfChangeReqSndDto.getByteArray();

        // 【ログ出力】
        // デバッグログを出力
        debugInfo = new LinkedHashMap<>();
        debugInfo.put("【送信電文データ出力】",
                String.format("LAN信号変換装置(%s) - 時間設定変更要求送信(LSC時間設定変更要求送信DTO)", IfLscControlParam.SEND_IPADDRESS));
        debugInfo.put("sndData", this.shapingByteList(sndData));

        logger.debug(callerClazz, "", debugInfo);

        //2.1. 電文送信処理を実行する。

        //2.1.1. 以下のJava標準モジュールを呼び出す。
        ISendTcpData sendTcpData = context.getBean(ISendTcpData.class);

        //2.1.2. 以下のJava標準モジュールを呼び出す。
        try {

            sendTcpData.sendTcpData(sndData);

            //2.2. 例外発生[IOException]の有無により以下を実行する。
        } catch (Exception e) {

            //2.2.1. 例外発生[IOException]が有の場合、以下を実行する。

            // 【ログ出力】
            // エラーログを出力
            debugInfo = new LinkedHashMap<>();

            logger.error(callerClazz, MsgId.EAE_03_00122, Arrays.asList("LAN信号変換装置"), debugInfo);

            //2.2.1.1. 電文実行結果DTO[messageResultDto]に以下の値を設定する。
            messageResultDto.resultCd = SystemCode.ResultIfCode.SEND_ERROR;

            //2.2.1.2. 戻り値として電文実行結果DTO[messageResultDto]を返却し、処理を終了する。
            return messageResultDto;

            //2.2.2. 上記以外の場合、何もしない。
        }

        // 要求に対する応答電文を受信するか、タイムアウトするまで待機する。
        LscResRcvDataDto lscResRcvDataDto =
                lscReqResReceiver.waitResponse(lscTimeConfChangeReqSndDto.comHeaderDto, null);
        messageResultDto = lscResRcvDataDto.messageResultDto;
        if (Objects.nonNull(lscResRcvDataDto.rcvData)) {
            BeanUtils.copyProperties(lscResRcvDataDto.rcvData, lscTimeConfChangeResRcvDto);
        }

        return messageResultDto;

    }

}
