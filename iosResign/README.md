### 重签名脚本 for ios app

内容介绍和使用见:
         [ios重签名脚本的0到1](https://paulswith.github.io/2017/11/14/ios%E9%87%8D%E7%AD%BE%E5%90%8D%E8%84%9A%E6%9C%AC%E7%9A%840%E5%88%B01/)
         
![](image/sign_profile.png)


使用
./resign.sh /Users/dobby/Desktop/微信-已砸壳.ipa  /Users/dobby/Desktop/original.mobileprovision  com.weixin.resign
$1 需要砸壳过的ipa,从PP助手下载即可
$2 .mobileprovision 这个是证书对应的文件,也可以指定证书Xcode的product下也会生成
$3 是否修改bundleID,默认是原先的


脚本内部需要写入一个一个签名字符串,可以通过下方的命令拿到
security find-identity -v -p codesigning

若是cryptid显示为0,非砸壳为1. 砸壳虽不影响签名的成功率, 但是我试了下可安装,但不可使用. 可google下砸壳方法自行砸壳.下面是查看cryptid的命令
otool -l ./腾讯手机管家-来电防骚扰的QQ安全助手\(正版\)/Payload/MQQSecure.app/MQQSecure | grep cryptid
mobileProvision生成Entitlements.plist
security cms -D -i $provision > ProvisionProfile.plist 
/usr/libexec/PlistBuddy -x -c "Print Entitlements" ProvisionProfile.plist > $tempPlace/Entitlements.plist
用该方法生成一个Entitlements.plist文件,之前还没找到这么快捷的生成方法, 有个土方法自己折腾出来的也是可行的,详见plist文件-在脚本中的操作

mobileProvision拷贝为embedded.mobileprovision
cp $provision $appPlace/embedded.mobileprovision
是否概要更改BundleID
if [[ $3 ]]; then
	plutil -replace CFBundleIdentifier -string "$3" $appPlace/Info.plist
	reBundleID=`plutil -p $appPlace/Info.plist | grep 'CFBundleIdentifier' `
	echo "Log info : you wanna replace to: ${reBundleID}"
fi
默认的时候是原先的, BundleID安装到设备上是唯一的,若是原版本共存,必须要更改

删除影响签名文件
plutil -remove CFBundleREsourceSpecification $appPlace/Info.plist   #删除签名源文件相关
rm -rf $appPlace/Watch  #发现watch插件必现失败,这个必须删除了
rm -rf $appPlace/PlugIns #发现PlugIns插件必现失败,这个必须删除了,就算下面重签也不管的, 坑超多
codesignInfo=`find $appPlace -name "CodeResources" `
for i in $codesignInfo; do
	rm -f $i
done
对Info.plist里面的CFBundleREsourceSpecification这个key删除,具体不知道做了什么,IosAppSigner也这么做了
Watch和Plugins这两个文件,我曾试过对他们都进行重签名, 但依然不管用, 相关坑大概是独立性,可以跳转到念茜女神的博文了解
源文件的_CodeSignature/CodeResources也进行了全局删除,签名后会自动生成这个文件
开始重签
第一步,相关lib-framework签名
allShouldSign=` find $appPlace -name "*.appex" && find $appPlace  -name "*.framework" && find $appPlace  -name "*.dylib" && find $appPlace/* -name "*.app" ` #最上层的先不签
for i in $allShouldSign; do
	codesign -fs "${signStr}" --no-strict --entitlements=/tmp/project_resign/Entitlements.plist $i
done
对上方类型都进行一篇搜索,得到绝对路径后进行重签名

这是iosAppResigner中,作者对着全部的类型都检索了一遍,我并完全覆盖,如果你踩到了坑,可以试试把类型都加上

核心包签名
codesign -vvv -fs "$signStr" --no-strict --entitlements=/tmp/project_resign/Entitlements.plist $appPlace
打包为ipa
cd $tempPlace
zip -qry sign.ipa ./Payload
mv $tempPlace/sign.ipa ~/Desktop
因为打包的时候是递归形式的, 指定绝对路径会踩坑,注意就行
