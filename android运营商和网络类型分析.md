# android运营商和网络类型分析
一些需求需要获取运营商和网络类型，下面对运营商和网络类型做分析。
先抛出一些废话的定义：

GSM：全球移动通讯系统Global System of Mobile communication就是众所周知的GSM，是当前应用最为广泛的移动电话标准。
CDMA：码分多址(CDMA)是在数字技术的分支--扩频通信技术上发展起来的一种崭新而成熟的无线通信技术。
可见，这两者是技术标准，和网络类型及制式无关。

##获取话机类型

这个可以通过方法TelephonyManager#getPhoneType来获得，下面是返回类型。
 
/**
     * Returns a constant indicating the device phone type.  This
     * indicates the type of radio used to transmit voice calls.
     *
     * @see #PHONE_TYPE_NONE
     * @see #PHONE_TYPE_GSM
     * @see #PHONE_TYPE_CDMA
     * @see #PHONE_TYPE_SIP
     */
常用话机类型就是GSM类型和CDMA类型，SIP是和VOIP相关的东西，平时不常遇到。
 
 
##获取运营商

TelephonyManager#getSimOperator用于获取SIM卡运营商ID，比如移动是46002
TelephonyManager#getSimOperatorName方法获取运营商名字，比如移动是CMCC
TelephonyManager#getSimCountryIso获取SIM卡国家，比如中国是cn
TelephonyManager#getSimState获取SIM卡状态
 
##获取网络类型
TelephonyManager#getNetworkType方法获取网络类型。
想要确切的显示出手机当前的网络，比如“联通3G”，需要的就是这个。
在网上找了一些代码，看见一些代码在一些网络类型后面标明：“移动2G”，我只想说“呵呵”。
原因就是，从单一的网络类型是无法判断这点的。
回到正题，开始分析返回值。
 
1）NETWORK_TYPE_GPRS

GPRS是一种制式，相当于2.5G，它独立于话机类型而存在，虽然移动是GSM话机，联通是CDMA话机，但是他们都可以有这种制式，
拿移动2G举例，我所在城市是EDGE网络。但是在之前，移动和联通可能有同时使用GPRS的时候，
同时也不排除部分地区移动仍然部署了GPRS的可能性，所以比较不赞同在代码后面标“移动2G”的这位前辈。
 
2）NETWORK_TYPE_EDGE

EDGE应该算是2.75G。据我所知，联通好像没有升级2G网络到这个制式。而移动当前是在用这个。
 
3）NETWORK_TYPE_UMTS

UMTS定义是一种3G移动电话技术，使用WCDMA作为底层标准，WCDMA向下兼容GSM网络。
目前中国也就只有联通了，这个确实可以唯一判断运营商及其网络类型。
 
4）NETWORK_TYPE_CDMA

CDMA的定义是一种技术标准，有其2代、2.5代、3代技术。被认为是3代移动技术的首选，包含的标准有
WCDMA、CDMA2000、TD-SCDMA。这里CDMA指代CDMA2代技术标准的制式，中国电信在用。
 
5）NETWORK_TYPE_1xRTT

在CDMA2000中，通常被认为是2.5G或2.75G，速率只有其他3G的几分之一，电信可能使用。
 
6）NETWORK_TYPE_EVDO_0、NETWORK_TYPE_EVDO_A、NETWORK_TYPE_EVDO_B

两者都是CDMA2000标准中的版本，属于3G，电信可能使用。
 
7）NETWORK_TYPE_HSDPA

一种通信协议，建立在WCDMA上，相当于3.5G，联通可能使用。
 
8）NETWORK_TYPE_LTE

对应准4G，各个运营商都可能使用。
 
9）NETWORK_TYPE_GSM

这个值是隐藏的，值为16，暂时不知道什么卡会出现。猜想应该是对应GSM标准的最早期制式，没有验证。
 
10）NETWORK_TYPE_TD_SCDMA

也是隐藏的，值为17，使用移动3G时是这个值。
 
结论：判断哪个运营商那种网络不应该只根据NetworkType判断。
运营商单独获取，而NetworkType可以进一步知道是2G还是3G。
其他中国不存在的制式就先不判断了。
 
##关于android版本兼容

对于android版本低的设备，不包含一些类型的定义，所以最好在自己的类中重新定义这些网络类型变量



