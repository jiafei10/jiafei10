/* ILscGateway.java.
 *
 * All rights reserved, Copyright(C) 2022 Oki Electric Industry Co.,Ltd.
 */

package com.oki.ofits.common.api.extif.lsc;

import org.springframework.stereotype.Component;

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
 * LAN信号変換装置Gatewayインタフェースクラス.
 * <p>
 * LAN信号変換装置のゲートウェイIFを提供する。
 * </p>
 */
@Component
public interface ILscGateway {

    /**
     * 時間応答送信メソッド
     * 
     * 無線回線制御装置に対し時刻を応答する。
     * @param lscTimeResSndDto LSC時間応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndTimeRes(LscTimeResSndDto lscTimeResSndDto);

    /**
     * 基地局選択状態要求送信メソッド
     * 
     * 指令台より無線通信を行う基地局を個別選択する。
     * @param lscBaseStaSelectStatusReqSndDto LSC基地局選択状態要求送信DTO
     * @param lscBaseStaSelectStatusResRcvDto LSC基地局選択状態応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndBaseStaSelectStatusReq(LscBaseStaSelectStatusReqSndDto lscBaseStaSelectStatusReqSndDto,
            LscBaseStaSelectStatusResRcvDto lscBaseStaSelectStatusResRcvDto);

    /**
     * LSC状態要求送信メソッド
     * 
     * 無線回線制御装置の状態を要求する。
     * @param lscStatusReqSndDto LSC状態要求送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscStatusReq(LscStatusReqSndDto lscStatusReqSndDto);

    /**
     * 規制制御状態要求送信メソッド
     * 
     * 無線回線制御装置に対し規制制御を要求する。
     * @param lscReglationCtrlStatusReqSndDto LSC規制制御状態要求送信DTO
     * @param lscReglationCtrlStatusResRcvDto LSC規制制御状態応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndReglationCtrlStatusReq(LscReglationCtrlStatusReqSndDto lscReglationCtrlStatusReqSndDto,
            LscReglationCtrlStatusResRcvDto lscReglationCtrlStatusResRcvDto);

    /**
     * 運用開始応答送信メソッド
     * 
     * 無線回線制御装置からの無線システムの運用開始通知を受け、運用開始要求結果を応答する。
     * @param lscOpeStartResSndDto LSC運用開始応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndOpeStartRes(LscOpeStartResSndDto lscOpeStartResSndDto);

    /**
     * LSC通信設定要求送信メソッド
     * 
     * 無線回線制御装置に対し通信設定を要求する。
     * @param lscCommConfReqSndDto LSC通信設定要求送信DTO
     * @param lscCommConfResRcvDto LSC通信設定応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommConfReq(LscCommConfReqSndDto lscCommConfReqSndDto,
            LscCommConfResRcvDto lscCommConfResRcvDto);

    /**
     * LSC通信開始要求送信メソッド
     * 
     * 無線回線制御装置に対し通信開始を要求する。
     * @param lscCommStartReqSndDto LSC通信開始要求送信DTO
     * @param lscCommStartResRcvDto LSC通信開始応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommStartReq(LscCommStartReqSndDto lscCommStartReqSndDto,
            LscCommStartResRcvDto lscCommStartResRcvDto);

    /**
     * LSC通信終了要求送信メソッド
     * 
     * 無線回線制御装置に対し通信終了を要求する。
     * @param lscCommEndReqSndDto LSC通信終了要求送信DTO
     * @param lscCommEndResRcvDto LSC通信終了応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscCommEndReq(LscCommEndReqSndDto lscCommEndReqSndDto,
            LscCommEndResRcvDto lscCommEndResRcvDto);

    /**
     * セレコール要求送信メソッド
     * 
     * 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
     * @param lscSelcallReqSndDto LSCセレコール要求送信DTO
     * @param lscSelcallResRcvDto LSCセレコール応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndSelcallReq(LscSelcallReqSndDto lscSelcallReqSndDto,
            LscSelcallResRcvDto lscSelcallResRcvDto);

    /**
     * チャネル占有通信設定要求送信メソッド
     * 
     * 無線回線制御装置に対し移動局発信の個別通信を受け付けたことを通知する。
     * @param lscChOccCommConfReqSndDto LSCチャネル占有通信設定要求送信DTO
     * @param lscChOccCommConfResRcvDto LSCチャネル占有通信設定応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndChOccCommConfReq(LscChOccCommConfReqSndDto lscChOccCommConfReqSndDto,
            LscChOccCommConfResRcvDto lscChOccCommConfResRcvDto);

    /**
     * 移動局在圏情報取得応答送信メソッド
     * 
     * 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
     * @param lscMobileStaAreaInfoGetResSndDto LSC移動局在圏情報取得応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndMobileStaAreaInfoGetRes(
            LscMobileStaAreaInfoGetResSndDto lscMobileStaAreaInfoGetResSndDto);

    /**
     * バージョン応答送信メソッド
     * 
     * 無線回線制御装置からのバージョン情報の通知要求を受け、バージョン情報を応答する。
     * @param lscVersionResSndDto LSCバージョン応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndVersionRes(LscVersionResSndDto lscVersionResSndDto);

    /**
     * LAN信号変換装置切替応答送信メソッド
     * 
     * 無線回線制御装置からの個別通信対象の移動局の在圏情報取得要求を受け、在圏情報を応答する。
     * @param lscSwitchResSndDto LSC切替応答送信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndLscSwitchRes(LscSwitchResSndDto lscSwitchResSndDto);

    /**
     * 時間設定変更要求送信メソッド
     * 
     * 無線回線制御装置に対しシステムの時間設定変更を要求する。
     * @param lsctimeConfChangeReqSndDto LSC時間設定変更要求送信DTO
     * @param lsctimeConfChangeResRcvDto LSC時間設定変更応答受信DTO
     * @return MessageResultDto 電文実行結果DTO
     */
    public MessageResultDto sndTimeConfChangeReq(LscTimeConfChangeReqSndDto lsctimeConfChangeReqSndDto,
            LscTimeConfChangeResRcvDto lsctimeConfChangeResRcvDto);

}
