title: IOS Recorder-lite APP 去广告
date: 2015-10-17 16:11:17
tags: [IOS越狱开发]
---
Recorder-lite是IOS上一个比较简洁的录音软件，有收费版本，免费的版本和收费版本区别就是广告。下面我们实战Recorder-lite的广告去除。<!-- more -->
首先看下广告的位置，我现在安装的版本为2.1.5版，广告如下：
![](/photo/rcdimg/adsview.png)
这个图片是在Reveal工具中截图出来的。可以看到，广告是在这个GADBannerView里。
用Clutch解密出程序，class-dump-z dump出头文件。找到GADBannerView.h文件。
{% codeblock lang:cpp %}
@interface GADBannerView : XXUnknownSuperclass <GADSlotAdEventDelegate, GADSlotDelegate> {
	BOOL _hasLoadedAd;
	BOOL _hasInitialized;
	BOOL _autoloadEnabled;
	UIViewController* _rootViewController;
	id<GADBannerViewDelegate> _delegate;
	id<GADInAppPurchaseDelegate> _inAppPurchaseDelegate;
	UIView* _rootAdView;
	GADSlot* _slot;
}
{% endcodeblock %} 
用类名检索引用，检索到以下几个文件，查看引用上下文，我们锁定了AdHelper.h。
![](/photo/rcdimg/class_sc.png)
AdHelper看名字看不出什么，但是打开一看可以发现他显然是一个广告管理的一个类。
{% codeblock lang:cpp %}
@interface AdHelper : XXUnknownSuperclass <GADInterstitialDelegate, ADInterstitialAdDelegate, ADBannerViewDelegate, GADBannerViewDelegate> {
	GADInterstitial* mobInterstitialAd_;
	ADInterstitialAd* interstitialAd_;
	DFPInterstitial* dfpInterstitialAd_;
	GADBannerView* boxAd_Amob;
	DFPBannerView* boxAd_Dfp;
	ADBannerView* boxAd_Apple;
	FPPopoverController* _popover;
	UIView* _contentView;
	UIView* _targetView;
	UIView* _boxAd;
	CGSize _oContentSize;
}
@property(assign, nonatomic) __weak UIView* boxAd;
@property(readonly, assign, nonatomic) BOOL isBoxAdLoaded;
...
+(id)sharedAdHelper;
-(void).cxx_destruct;
-(void)clearContentSubViews;
-(void)layoutForCurrentOrientation;
-(void)showFullAd;
-(void)createFullAd;
-(void)createAppleFullAd;
-(void)createAdmobFullAd;
-(void)createDFPFullAd;
-(void)createBoxAd;
...
@end
{% endcodeblock %} 
从类的成员函数我们可以顾名思义，showFullAd应该是现实所有的广告，createFullAd创建所有广告。以及一些创建广告的方法。使用Hopper查看：
{% codeblock lang:cpp %}
void -[AdHelper createFullAd](void * self, void * _cmd) {
    [self createBoxAd];
    [self createAdmobFullAd];
    [self createAppleFullAd];
    Pop();
    Pop();
    Pop();
    r0 = loc_187474(self, @selector(createDFPFullAd));
    return;
}
{% endcodeblock %} 
作者挺为我们考虑的，这里应该是初始化广告的函数了。直接建个tweak工程键入代码，为了简单，我hook了他所有创建ad的函数：
{% codeblock lang:cpp %}
@interface AdHelper {

}
-(void)showFullAd;
-(void)createFullAd;
-(void)createAppleFullAd;
-(void)createAdmobFullAd;
-(void)createDFPFullAd;
-(void)createBoxAd;
@end

%hook AdHelper
-(void)showFullAd{return;}
-(void)createFullAd{return;}
-(void)createAppleFullAd{return;}
-(void)createAdmobFullAd{return;}
-(void)createDFPFullAd{return;}
-(void)createBoxAd{return;}
%end
{% endcodeblock %} 
编译发送到手机测试，发现广告已经不加载了：
![](/photo/rcdimg/pass_ads.png)
不过我们看到，录音机上边有几个按钮，两个购物车按钮，一个下载的按钮。一个是wifi web连接。还有一个是mp3格式切换的按钮。虽然广告不加载了，但是如果点击这些按钮下载购物车按钮，会跳转到appstore里去下载软件。这样作为强迫症的我们是不能接受的。下面我们来分析隐藏一下这些按钮。
我们在hopper里搜索itunes，找到练到itunes的连接，根据引用查找到想过的代码，
{% codeblock lang:cpp %}
 XREF=-[RootViewController reviewAction]+88
 XREF=-[RootViewController moreApp]+326
 XREF=-[RecordingViewController buyAction]+72, -[RootViewController cloud:]+72
 {% endcodeblock %} 
 总共找到4处引用，涉及RootViewController和RecordingViewController两个类。
 {% codeblock lang:cpp %}
 @interface RecordingViewController : XXUnknownSuperclass <RecorderDelegate> {
	double lowPassResults_;
	UIButton* recordOrPauseButton;
	UIButton* listOrStopButton;
	UIImageView* timeBackground;
	UILabel* timeLable;
	UIButton* formatButton;
	Recorder* recorder;
	NSTimer* levelTimer_;
	int count;
	UIButton* buyButton;
	id<RecordingViewControllerDelegate> _delegate;
	UIView* _firstResponderView;
}
@interface RootViewController : XXUnknownSuperclass <FPPopoverControllerDelegate, FilesViewControllerDelegate, RecordingViewControllerDelegate, PlayingViewControllerDelegate> {
	FilesViewController* filesViewController;
	UIView* lineView;
	UIButton* cloudButton;
	UIButton* recordingButton;
	UIButton* wifiButton;
	UIView* emptyView;
	UIImageView* emptyImageView;
	UILabel* emptyLabel;
	FPPopoverController* popover;
	RecordingViewController* recordingViewController;
	PlayingViewController* playingViewController;
	WiFiViewController* wiFiViewController;
	UIView* handView;
	UIButton* downloadButton;
	UIImageView* imageN;
}
{% endcodeblock %} 
对比程序界面，我们发现了几个可疑的UIButton：buyButton、cloudButton、downloadButton，我们试试在界面加载完成的时候隐藏掉。修改tweak代码：
{% codeblock lang:cpp %}
#import <UIkit/UIkit.h>
#import <RootViewController.h>
#import <AdHelper.h>
#import <RecordingViewController.h>

%hook AdHelper
-(void)showFullAd{return;}
-(void)createFullAd{return;}
-(void)createAppleFullAd{return;}
-(void)createAdmobFullAd{return;}
-(void)createDFPFullAd{return;}
-(void)createBoxAd{return;}
%end

%hook RootViewController
-(void)viewDidLoad
{
	%orig;
	UIButton* downloadButton = MSHookIvar<UIButton* >(self,"downloadButton");
	downloadButton.hidden = YES;
	UIButton* cloudButton = MSHookIvar<UIButton* >(self,"cloudButton");
	cloudButton.hidden = YES;
}
%end

%hook RecordingViewController
-(void)viewDidLoad
{
	%orig;
	UIButton* buyButton = MSHookIvar<UIButton* >(self,"buyButton");
	buyButton.hidden = YES;
}
%end
{% endcodeblock %} 
我们惊喜的发现RootViewController上边的按钮被顺利隐藏了，但是RecordingViewController上的按钮却没被隐藏。
![](/photo/rcdimg/pass_ads_a.png)
经过测试发现，隐藏buyButton的时候buyButton还没被创建。在录音按钮响应函数里加入隐藏代码，发现可以生效，而每次加载这个录音页面的时候，这个按钮会重新加载进来，所以这个按钮应该是动态创建的。我们在hopper里搜索buyButton。
{% codeblock lang:cpp %}
001fdb9b         db  0x00 ; '.'
             objc_ivar_offset_RecordingViewController_buyButton:
001fdb9c         db  0xcc ; '.'                                                 ; XREF=-[RecordingViewController dealloc]+50, -[RecordingViewController dealloc]+52, -[RecordingViewController viewDidAppear:]+58, -[RecordingViewController viewDidAppear:]+60, -[RecordingViewController .cxx_destruct]+16, -[RecordingViewController .cxx_destruct]+18

void -[RecordingViewController viewDidAppear:](void * self, void * _cmd, char arg2) {
	......
	r8 = *objc_ivar_offset_RecordingViewController_buyButton;
	r0 = [UIButton buttonWithType:0x0];
	r0 = [r0 retain];
	r1 = *(r4 + r8);
	*(r4 + r8) = r0;
	[r1 release];
	r5 = *(r4 + r8);
	r6 = [[UIImage imageNamed:@"buy.png"] retain];
	[r5 setBackgroundImage:r6 forState:0x0];
	[r6 release];
	......
}
{% endcodeblock %} 
动态加载buyButton的代码在RecordingViewController viewDidAppear函数里，我们hook掉他试试。
{% codeblock lang:cpp %}
#import <UIkit/UIkit.h>
#import <RootViewController.h>
#import <AdHelper.h>
#import <RecordingViewController.h>

%hook AdHelper
-(void)showFullAd{return;}
-(void)createFullAd{return;}
-(void)createAppleFullAd{return;}
-(void)createAdmobFullAd{return;}
-(void)createDFPFullAd{return;}
-(void)createBoxAd{return;}
%end

%hook RootViewController
-(void)viewDidLoad
{
	%orig;
	UIButton* downloadButton = MSHookIvar<UIButton* >(self,"downloadButton");
	downloadButton.hidden = YES;
	UIButton* cloudButton = MSHookIvar<UIButton* >(self,"cloudButton");
	cloudButton.hidden = YES;
}
%end

%hook RecordingViewController
-(void)viewDidAppear:(BOOL)view
{
	%orig;
	UIButton* buyButton = MSHookIvar<UIButton* >(self,"buyButton");
	buyButton.hidden = YES;
}
%end
{% endcodeblock %} 
![](/photo/rcdimg/pass_ads_end.png)
我们的app界面已经相当的简洁了。