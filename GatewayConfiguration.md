 * LscGatewayConfiguration.java.
 *
 * All rights reserved, Copyright(C) 2022 Oki Electric Industry Co.,Ltd.
 */

package com.oki.ofits.common.api.extif.lsc;

import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.IntegrationFlows;
import org.springframework.integration.gateway.GatewayProxyFactoryBean;
import org.springframework.integration.ip.dsl.Tcp;
import org.springframework.integration.ip.dsl.TcpClientConnectionFactorySpec;
import org.springframework.integration.ip.tcp.serializer.TcpCodecs;

import com.oki.ofits.common.controlparam.SystemControlParam.IfLscControlParam;

/**
 * LAN信号変換装置GatewayConfigurationクラス.
 * <p>
 * LAN信号変換装置Gatewayクラスの設定を提供する。
 *
 * </p>
 */
@Configuration
@EnableIntegration
public class LscGatewayConfiguration {

    private LscRcvGateway lscRcvGateway;

    @Autowired
    public LscGatewayConfiguration(LscRcvGateway lscRcvGateway) {

        this.lscRcvGateway = lscRcvGateway;
    }

    @ServiceActivator(inputChannel = "lscInboundTcpChannel")
    public void process(byte[] message) {

        lscRcvGateway.rcvData(message);

        return;

    }

    /**
     * データ送信インタフェース(LAN信号変換(TCP))
     * 
     * LAN信号変換装置Gatewayクラスのデータ送信処理(LAN信号変換(TCP))を定義する。
     * 
     * @param byte[]
     * 
     * @return void
     */
    public static interface ISendTcpData {

        void sendTcpData(byte[] sendDeta);

    }

    /**
     * アウトバウンドチャンネル(LAN信号変換(TCP))メソッド
     * 
     * LAN信号変換装置Gatewayクラスのクライアントチャンネル設定(LAN信号変換(TCP))を定義する。
     * @return DirectChannel 
     */
    @Bean
    public DirectChannel lscOutboundTcpChannel() {

        //1. ダイレクトチャンネルを生成し、戻り値として返却し処理を終了する。
        return new DirectChannel();

    }

    /**
     * クライアントコネクション(指令制御(TCP))メソッド
     * 
     * 指令制御装置Gatewayクラスのクライアントコネクション設定(指令制御(TCP))を定義する。
     * @return DirectChannel 
     */
    @Bean
    public TcpClientConnectionFactorySpec lscTcpClientConnectionFactorySpec() {

        TcpClientConnectionFactorySpec tcpClientConnectionFactorySpec =
                Tcp.nioClient(IfLscControlParam.SEND_IPADDRESS, IfLscControlParam.SEND_PORT_NO)
                        .serializer(TcpCodecs.raw())
                        .deserializer(message -> lscOutboundTcpDeserializer(message));

        return tcpClientConnectionFactorySpec;
    }

    /**
     * アウトバウンドフロー(LAN信号変換(TCP))メソッド
     * 
     * LAN信号変換装置Gatewayクラスのアウトバウンドのフローを定義する。
     * @return IntegrationFlow 
     */
    @Bean
    public IntegrationFlow lscOutboundTcpFlow() {
        return IntegrationFlows
                .from(lscOutboundTcpChannel()) // lscOutboundTcpChannelからの入力を受け付ける
                .handle(Tcp.outboundAdapter(lscTcpClientConnectionFactorySpec())) // LSCへ電文を送信する
                .get();
    }

    /**
     * インバウンドフロー(指令制御(TCP))メソッド
     * 
     * LAN信号変換装置GatewayクラスのインバウンドのTCPフローを定義する。
     * @return IntegrationFlow 
     */
    @Bean
    public IntegrationFlow lscInboundTcpFlow() {
        return IntegrationFlows
                .from(Tcp.inboundAdapter(lscTcpClientConnectionFactorySpec())) // LSCから電文を受信する
                .channel("lscInboundTcpChannel") // lscInboundTcpChannelへ出力する（LSCRcvGatewayに処理を委託）
                .get();
    }

    /**
     * アウトバウンドファクトリ(LAN信号変換(TCP))メソッド
     * 
     * LAN信号変換装置GatewayクラスのTCP接続ファクトリ設定を定義する。
     * @return GatewayProxyFactoryBean 
     */
    @Bean
    public GatewayProxyFactoryBean lscOutboundTcpFactory() {

        //1. ゲートウェイ[gateway]を生成する。
        GatewayProxyFactoryBean gateway = new GatewayProxyFactoryBean(ISendTcpData.class);

        //2. 以下のJava標準モジュールを呼び出す。
        gateway.setDefaultRequestChannel(lscOutboundTcpChannel()); // lscOutboundTcpChannelへデータを出力する

        //3. 戻り値としてゲートウェイ[gateway]を返却し、処理を終了する。
        return gateway;

    }

    /**
     * アウトバウンドデシリアライザ(LAN信号変換(TCP))メソッド
     * 
     * LAN信号変換装置GatewayクラスのTCPフローにおけるデシリアライザを行う。
     * @param streamData ストリームデータ
     * @return byte[] バイト配列
     */
    public byte[] lscOutboundTcpDeserializer(InputStream streamData) {

        //1. 以下の変数を生成する。
        byte[] ｍsgSizeCheckArray = new byte[6];

        //2. 以下のJava標準モジュールを呼び出す。
        try {
            streamData.read(ｍsgSizeCheckArray, 0, 6);
        } catch (IOException e) {
            e.printStackTrace();
        }

        //3. 以下の変数を設定する。
        int msgSize = 0;
        try {
            msgSize = Integer.parseInt(new String(ｍsgSizeCheckArray, "MS932").substring(2, 6));
        } catch (NumberFormatException e1) {
            // TODO 自動生成された catch ブロック
            e1.printStackTrace();
        } catch (UnsupportedEncodingException e1) {
            // TODO 自動生成された catch ブロック
            e1.printStackTrace();
        }

        //4. 以下の変数を設定する。
        byte[] replyMsgArray = new byte[msgSize + 6];

        //5. 以下のJava標準モジュールを呼び出す。
        System.arraycopy(ｍsgSizeCheckArray, 0, replyMsgArray, 0, ｍsgSizeCheckArray.length);

        //6. 以下のJava標準モジュールを呼び出す。
        try {
            streamData.read(replyMsgArray, ｍsgSizeCheckArray.length, msgSize);
        } catch (IOException e) {
            e.printStackTrace();
        }

        //7. 戻り値として応答メッセージ配列[replyMsgArray]を返却して、処理を終了する。
        return replyMsgArray;

    }

}
