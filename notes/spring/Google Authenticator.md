# 双因素认证

## Google Authenticator
Google Authenticator 使用了一种基于**时间**的 TOTP 算法, 其中时间的选取为自 1970-01-01 00:00:00 以来的毫秒数除以 30 与 客户端及服务端约定的**密钥**进行计算, 计算结果为一个**6位数的字符串**( 首位数可能为0, 所以为字符串), 所以在 Google Authenticator 中我们可以看见验证码每个 30 秒就会刷新一次

## 实现思路
生成验证码有俩个重要的参数, 其一为**客户端与服务端约定的密钥**, 其二便为**30秒的个数**

```java
/**
 * 随机生成一个密钥
 */
public static String createSecretKey() {
    SecureRandom random = new SecureRandom();
    byte[] bytes = new byte[20];
    random.nextBytes(bytes);
    Base32 base32 = new Base32();
    String secretKey = base32.encodeToString(bytes);
    return secretKey.toLowerCase();
}
```
```java
//1970-01-01 00:00:00 以来的毫秒数除以30 
long time = System.currentTimeMillis() / 1000 / 30;
```

根据这两个参数就可以生成一个验证码
```java
/**
 * 根据密钥获取验证码
 * 返回字符串是因为验证码有可能以 0 开头
 * @param secretKey 密钥
 * @param time      第几个 30 秒 System.currentTimeMillis() / 1000 / 30
 */
public static String getTOTP(String secretKey, long time) {
    Base32 base32 = new Base32();
    byte[] bytes = base32.decode(secretKey.toUpperCase());
    String hexKey = Hex.encodeHexString(bytes);
    String hexTime = Long.toHexString(time);
    return TOTP.generateTOTP(hexKey, hexTime, "6");
}
```

因为 Google Authenticator 计算验证码也需要**密钥**的参与, 而时间则会在本地获取, 所以我们需要将**密钥保存起来**, 同时为了与其他账户进行区分, 除了密钥外, 我们还需要录入**服务名称 , 用户账户**信息. 而为了方便用户信息的录入, 我们一般将所有信息生成一张二维码图片, 让用户通过扫码自动填写相关信息
```java
/**
 * 生成 Google Authenticator 二维码所需信息
 * Google Authenticator 约定的二维码信息格式 : otpauth://totp/{issuer}:{account}?secret={secret}&issuer={issuer}
 * 参数需要 url 编码 + 号需要替换成 %20
 * @param secret  密钥 使用 createSecretKey 方法生成
 * @param account 用户账户 如: example@domain.com 138XXXXXXXX
 * @param issuer  服务名称 如: Google Github 印象笔记
 */
public static String createGoogleAuthQRCodeData(String secret, String account, String issuer) {
    String qrCodeData = "otpauth://totp/%s?secret=%s&issuer=%s";
    try {
        return String.format(qrCodeData, URLEncoder.encode(issuer + ":" + account, "UTF-8").replace("+", "%20"), URLEncoder.encode(secret, "UTF-8")
                .replace("+", "%20"), URLEncoder.encode(issuer, "UTF-8").replace("+", "%20"));
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }
    return "";
}
```

让用户输入验证码, 与服务端进行校验, 如果校验通过, 则表明用户可以完好使用该功能

因为验证码是使用基于时间的 TOTP 算法, 依赖于客户端与服务端时间的一致性. 如果客户端时间与服务端时间相差过大, 那在用户没有同步时间的情况下, 永远与服务端进行匹配. 同时服务端也有可能出现时间偏差的情况, 这样反而导致时间正确的用户校验无法通过

为了解决这种情况, 我们可以使用**时间偏移量**来解决该问题, Google Authenticator 验证码的时间参数为 1970-01-01 00:00:00 以来的毫秒数除以 30, 所以每 30 秒就会更新一次. 但是我们在后台进行校验时, 除了与当前生成的二维码进行校验外, 还会对当前时间参数**前后偏移量**生成的验证码进行校验, 只要其中任意一个能够校验通过, 就代表该验证码是有效的
```java
/** 时间前后偏移量 */
private static final int timeExcursion = 3;
/**
 * 校验方法
 * @param secretKey 密钥
 * @param code      用户输入的 TOTP 验证码
 */
public static boolean verify(String secretKey, String code) {
    long time = System.currentTimeMillis() / 1000 / 30;
    for (int i = -timeExcursion; i <= timeExcursion; i++) {
        String totp = getTOTP(secretKey, time + i);
        if (code.equals(totp)) {
            return true;
        }
    }
    return false;
}
```

## 实例代码
https://github.com/ghthou/Google-Authenticator.git
