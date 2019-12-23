# 云闪付二维码实现自动收款接口思路整理 20191220

## 为什么使用云闪付
1、云闪付个人码收款T0直接到银行卡，非常方便，而且0费率，省去提现环节
2、云闪付风控比较低，单账号每日1000笔 3w额度几乎不触发异常,而且每人可以开通2个云闪付账号
3、云闪付app支持所有银联app扫码支付，用户也比较方便，免签约

## 系统时序设计
### 二维码生成
1、客户向服务器发起支付请求
2、服务器推送支付请求到QR任务APP
3、QR任务APP根据客户支付到额度，调用云闪付APP生成收款二维码，回传给服务器，直接显示给客户

### 用户付款
1、用户扫码完成支付
2、QR任务APP监控云闪付收款账单API，获得收款成功后立即把已到账金额传回给服务器
3、服务器收到已到账金额，根据金额匹配客户订单，通知客户完成支付

## 系统并发问题
1、每个云闪付生成二维码需要有1秒以上间隔，实测如果生成二维码频率过快，会导致云闪付APP崩溃，可以采用多个云闪付app轮训处理
2、如果客户支付频率过快，等额度订单过多，会导致订单错乱，无法根据支付额度回调，可以采用浮动订单金额来区分订单，例如100.01,100.02
3、云闪付APP收款后推送不及时，或者漏掉，可以使用HTTPS抓包方式，获取云闪付APP中收款记录的接口刷新来获取收款记录
4、收款手机闪断、会导致出码，收款环节异常，影响成功率，可以采用云手机或者模拟器方式来运行APP，提高app稳定性



## 技术关键实现
### xposed 调用云闪付生成二维码
```
public static void GenQrCode(final String money, final String mark,final String tid) {
        new Thread(()-> {
                try {
                    String mark1 = mark;
                    String money1 = new BigDecimal(money).setScale(2, RoundingMode.HALF_UP).toPlainString().replace(".", "");
                    LogeUtil.d("FORRECODE GenQrCode:0 money:" + money1 + " mark:" + mark1);
                    String str2 = "https://pay.95516.com/pay-web/restlet/qr/p2pPay/applyQrCode?txnAmt=" + Enc(money1) + "&cityCode=" + Enc(getcityCd()) + "&comments=" + Enc(mark1) + "&virtualCardNo=" + encvirtualCardNo;
                    OkHttpClient client = new OkHttpClient();
                    Request request = new Request.Builder()
                            .url(str2)
                            .header("X-Tingyun-Id", getXTid())
                            .header("X-Tingyun-Lib-Type-N-ST", "0;" + System.currentTimeMillis())
                            .header("sid", getSid())
                            .header("urid", geturid())
                            .header("cityCd", getcityCd())
                            .header("locale", "zh-CN")
                            .header("User-Agent", "Android CHSP")
                            .header("dfpSessionId", getDfpSessionId())
                            .header("gray", getgray())
                            .header("key_session_id", "")
                            .header("Host", "pay.95516.com")
                            .build();
                    Response response = client.newCall(request).execute();
                    /*LogeUtil.d("重试生成二维码==》11111");
                    Message message0 = new Message();
                    message0.what = 2;
                    message0.obj = money + "-" + mark + "-" + tid;
                    handler.sendMessageDelayed(message0, 1000);*/
                    if(response==null || response.body()==null || response.body().equals("null") || response.equals("null")){
                        error_times++;
                        if(error_times<20){
                            LogeUtil.d("重试生成二维码==》11111");
                            Message message = new Message();
                            message.what = 2;
                            message.obj = money + "-" + mark + "-" + tid;
                            handler.sendMessageDelayed(message, 3000);
                            LogeUtil.d("qr获取错误，3秒后重试==》11111");
                        }else{
                            LogeUtil.d("连续错误100次，无法获取二维码");
                            //关闭云闪付接单
                            sendStopYunShanBroad();
                        }
                    }
                } catch (Throwable e) {
                    error_times++;
                    if(error_times<20){
                        LogeUtil.d("重试生成二维码==》33333");
                        Message message = new Message();
                        message.what = 2;
                        message.obj = money + "-" + mark + "-" + tid;
                        handler.sendMessageDelayed(message, 3000);
                        LogeUtil.d("生成二维码异常==》"+e.getMessage());
                        LogeUtil.d("qr获取错误，3秒后重试==》33333");
                    }else{
                        LogeUtil.d("连续错误100次，无法获取二维码");
                        //关闭云闪付接单
                        sendStopYunShanBroad();
                    }
                }
        }).start();
    }
```
### xposed 调用云闪付获取收款金额
```
 public static void getPaidList()
    {
        final String str2 = "https://wallet.95516.com/app/inApp/order/list?currentPage=" + Enc("1") + "&month=" + Enc("0") + "&orderStatus=" + Enc("0") + "&orderType=" + Enc("A30000") + "&pageSize=" + Enc("10") + "";
        OkHttpClient client = null;
        OkHttpClient.Builder builder = new OkHttpClient().newBuilder();
        builder.connectTimeout(50, TimeUnit.SECONDS);
        builder.writeTimeout(50, TimeUnit.SECONDS);
        builder.readTimeout(50, TimeUnit.SECONDS);
        client = builder.build();
        final Request request = new Request.Builder()
                .url(str2)
                .header("X-Tingyun-Id", getXTid())
                .header("X-Tingyun-Lib-Type-N-ST", "0;" + System.currentTimeMillis())
                .header("sid", getSid())
                .header("urid", geturid())
                .header("cityCd", getcityCd())
                .header("locale", "zh-CN")
                .header("User-Agent", "Android CHSP")
                .header("dfpSessionId", getDfpSessionId())
                .header("gray", getgray())
                .header("Accept", "*/*")
                .header("key_session_id", "")
                .header("Host", "wallet.95516.com")
                .build();
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                LogeUtil.d("httperror:"+str2);
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                String RSP = response.body().string();
                //LogeUtil.d("CheckNewOrder str2=>" + str2 + " RSP=>" + RSP);
                String DecRsp = Dec(RSP);
                //LogeUtil.d("CheckNewOrder str2=>" + str2 + " DecRSP=>" + DecRsp);
                //这里有很多笔，可以自己调整同步逻辑s
                if( DecRsp!=null && !DecRsp.equals("") )
                {
                    JSONArray o = null;
                    try {
                        o = new JSONObject(DecRsp).getJSONObject("params").getJSONArray("uporders");
                        for (int i = 0; i < o.length(); i++) {
                            JSONObject p = o.getJSONObject(i);
                            //LogeUtil.d(p.toString());
                            String orderId = p.getString("orderId");
                            String amount = p.getString("amount");
                            String title = p.getString("title");
                            String orderTime = p.getString("orderTime");
                            Intent intent=new Intent(Constants.app_order_rec);
                            intent.putExtra("orderId",orderId);
                            intent.putExtra("amount",amount);
                            intent.putExtra("title",title);
                            intent.putExtra("orderTime",orderTime);
                            //intent.putExtra("isFristOrder",true);
                            intent.putExtra("isFrist",true);
                            if (app == null) {
                                getContext().sendBroadcast(intent);
                            } else {
                                app.sendBroadcast(intent);
                            }
                        }

                        Intent intent=new Intent(Constants.app_order_rec);
                        intent.putExtra("isFrist",false);//将状态设置不是首次
                        intent.putExtra("needCatch",false);//将状态设置不是首次
                        if (app == null) {
                            getContext().sendBroadcast(intent);
                        } else {
                            app.sendBroadcast(intent);
                        }

                    }catch (JSONException e)
                    {
                        LogeUtil.d(e.getMessage());
                    }
                }
            }
        });
    }
```
### xposed 屏蔽云闪付版本更新、隐藏root、隐藏模拟器信息
```
XposedHelpers.findAndHookMethod(
                    "com.unionpay.network.model.UPUpdateInfo",
                    loadPackageParam.classLoader,
                    "getUpdateCode", new Object[]{new XC_MethodHook() {
                        @Override
                        protected void afterHookedMethod(MethodHookParam param) throws Throwable { 
                            XposedBridge.log("屏蔽升级成功");
                        }
                    }});
XposedHelpers.findAndHookMethod(
                    "com.unionpay.utils.o",
                    loadPackageParam.classLoader,
                    "n", new Object[]{new XC_MethodHook() {
                        @Override
                        protected void afterHookedMethod(MethodHookParam param) throws Throwable { 
                            XposedBridge.log(" root result :" + param.getResult());
                        }
                    }});
XposedHelpers.findAndHookMethod(
                    "com.unionpay.utils.o",
                    loadPackageParam.classLoader,
                    "m", new Object[]{new XC_MethodHook() {
                        @Override
                        protected void afterHookedMethod(MethodHookParam param) throws Throwable { 
                            XposedBridge.log(" root result :" + param.getResult());
                        }
                    }});
```

## 总结
云闪付收款已经非常完善了，平均2-3个月一次风控调整，依然比较稳定。
但是也有许多细节需要改进，欢迎大家一起学习讨论  wx:luck_guoxiaonian
