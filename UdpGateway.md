 * TcpGateway.java.
 *
 * All rights reserved, Copyright(C) 2022 Oki Electric Industry Co.,Ltd.
 */

package com.oki.ofits.common.api.extif;

import java.util.Set;

import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;

/**
 * TCPGatewayクラス.
 * <p>
 * TCP-Gatewayの共通処理を提供する。
 *
 *  ① 送受信データの検証を実行する。
 * </p>
 */
public class UdpGateway {

    /**
     * データ検証メソッド
     * 
     * 送受信データの検証を実行する。
     * @param <T> Generics
     * @param data 検証対象データ
     * @return Set<ConstraintViolation<T>> 検証結果
     */
    protected static <T> Set<ConstraintViolation<T>> validateData(T data) {

        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        Set<ConstraintViolation<T>> constraintViolations = validator.validate(data);

        // 検証対象データを検証
        //        Set<ConstraintViolation<T>> constraintViolations =
        //                Validation.buildDefaultValidatorFactory().getValidator().validate(data);

        //2. 戻り値として検証結果[constraintViolations]を返却して、処理を終了する。
        return constraintViolations;

    }

    /**
     * バイト配列整形メソッド
     * 
     * バイト配列を16進数として見やすい形に加工する。
     * @param data 変換対象データ
     * @return String 整形後文字列
     */
    protected String shapingByteList(byte[] data) {

        StringBuffer sb = new StringBuffer();
        
        for (int i = 0; i < data.length; i++) {
            
            sb.append(String.format("%02X ", Byte.toUnsignedInt(data[i])));
            
            if (i % 16 == 15) {
                
                sb.append("| ");
                
            }
            
        }
        
        return sb.toString();

    }

}
