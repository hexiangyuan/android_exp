# ezdelivery Android 开发具体文档

### Tips
1. 正式库和测试库url

    * url 的地址在AppUr.java中， 全部使用的webApi的形式；api 的THRIFT文档对应的是ApiDoc中的Delivery.thirft.

    ```
    public static String kJsonRpcCoreUrl = BuildConfig.DEBUG ? "http://delivery.65emall.net/api/" : "http://pdt.65daigou.com/api/";
    ```

2. 自动更新
        * 使用的是蒲公英Sdk ，分别在LoginActivity 和 MainActivity 中注册；
        * 更新判断是y依据是build.gradle的versionCode；
        * 更新注册dialog在 PgyManager.java 中自定义； 

3.  扫描相关

    * 所有的扫描界面都是extends zxing中的CaptureActivity.java();下面有三个RadioButton 分别是Shelf，Parcel，pick up（默认是不显示的，只有在"签名Sign"的时候被调用）；

    * scan to shelf （扫描上架） ScanActivity  extends CaptureActivity；

    * out for delivery（配送）OutForDeliveryScanActivity extend ScanActivity；

    * sign （签名） SignScanActivity extend ScanActivity； (默认选中 pick up RadioButton)；

    * Picking - scan  ScanParcel2BoxActivity extends ScanActivitity；



### 登陆界面

1. 所有信息都保存在sp中，未考虑到信息安全，包括用户名，密码，登陆国家，cookie，UserType；

    * 通过RadioGroup的切换，来调用接口list<TServer> GetServers(1: string mode),来获取不同的Url；

    * 登陆成功时候会保存所有信息到SP中；

    * 保存的信息到Sp中的有
```java
   editor.putString("username", user);
        editor.putString("password", pwd);
        editor.putString("serverUrl", serverUrl);
        editor.putString("country", country);
        editor.putString("token", tLoginResult.token);
        editor.putBoolean("mode", AppUrl.isLiving());
        editor.putInt("stationSize", tLoginResult.StationNames.size());
        for (int i = 0; i < tLoginResult.StationNames.size(); i++) {
            editor.putString("stationName_" + i, tLoginResult.StationNames.get(i));
        }
```
* 保存到EzDelivreyApplication的static  参数

```java
SelfStationApplication.getInstance().setLoginResult(response);
```
*  TLoginResult的参数含义以及使用场景

```java
 public class TLoginResult extends BaseModule<TLoginResult> implements Serializable {
        public boolean isSuccessful;
        public String token; //cookie
        public java.util.ArrayList<String> StationNames;        //ParcelListActivity 这个界面的Menu中会用到stations
        public String userType;     //mainActivity界面的Button的显示和隐藏
    }
```
        
```java
public class TLoginResult extends BaseModule<TLoginResult> implements Serializable {
        public boolean isSuccessful;
        public String token; //cookie
        public java.util.ArrayList<String> StationNames;        //ParcelListActivity 这个界面的Menu中会用到stations
        public String userType;     //mainActivity界面的Button的显示和隐藏
    }
```

* **UserType 这个参数对应的是登陆人员的类型 目前只有 delivery staff  和 partner shop 两种，这两种身份在进入主界面是相应的控制显示与隐藏的Button**

        ```
        对应Tower内容https://tower.im/projects/fe05e38047f94e8694dc1bf61894f060/todos/c515e57a64814119b45dc243f1af5c6f/
        ```

###主页界面

 * 只是各个功能点Button以及所对应的入口。

 * 具体的显示与隐藏按钮的代码
```java
        if (CountryInfo.isSingapore()) {
            cdParcelList.setVisibility(View.GONE);
            cdSign.setVisibility(View.GONE);
            cdPartnerShop.setVisibility(View.GONE);
            }
        if (CountryInfo.isMalaysia() && LoginManager.isPartnerShop()) {
            cdScanToShelf.setVisibility(View.GONE);
            cdPicking.setVisibility(View.GONE);
            cdOut4Delivery.setVisibility(View.GONE);
            cdReady4Collection.setVisibility(View.VISIBLE);
            cdSign.setVisibility(View.VISIBLE);
            cdSearchParcel.setVisibility(View.VISIBLE);
            cdParcelList.setVisibility(View.VISIBLE);
            cdPartnerShop.setVisibility(View.VISIBLE);
            cdSetting.setVisibility(View.VISIBLE);
        }else if(CountryInfo.isMalaysia()&&LoginManager.isDeliveryStaff()){
            cdScanToShelf.setVisibility(View.VISIBLE);
            cdPicking.setVisibility(View.VISIBLE);
            cdOut4Delivery.setVisibility(View.VISIBLE);
            cdReady4Collection.setVisibility(View.VISIBLE);
            cdSign.setVisibility(View.VISIBLE);
            cdSearchParcel.setVisibility(View.VISIBLE);
            cdParcelList.setVisibility(View.GONE);
            cdPartnerShop.setVisibility(View.GONE);
            cdSetting.setVisibility(View.VISIBLE);
        }
```

* 蒲公英自动更新SDK注册；    

### **上架/scan to shelf**

#### **上架和二次上架以及Ready for collection 所对应的共同视图是ScanActivity**

* 扫描时候回扫描一个Shelf 和Parcel 然后调用接口DeliveryService.UserScanToShelf向服务器请求验证；
返回值Response **当且仅当response是""才是正常** 可以继续扫描，当response不为空的的时候就会出现错误，Alert错误信息；

* ScanActivity 是Scan To Shelf 和 Ready For Collection共用的界面，在共用的时候通过参数**from**区分
```java
Intent scanner = new Intent(this, ScanActivity.class);
scanner.putExtra("from", ScanActivity.SCAN_TO_SHELF);
startActivity(scanner);

 Intent ready4Collection = new Intent(this, ScanActivity.class);
ready4Collection.putExtra("from", ScanActivity.READY_FOR_COLLECTION);
tartActivity(ready4Collection);
```

* Scan to shelf 和 Ready for collection 都是通过扫描Shelf 和Parcel 调用接口和服务器接口给区分；
```java
  switch (from) {
    case READY_FOR_COLLECTION:
        Log.d("ScanActivity", "Ready for Collection:" + code);
        readyForCollection(code);
    break;

    case SCAN_TO_SHELF:
        Log.d("ScanActivity", "Scan to shelf:" + code);
        scanToShelf(code) ;
     break;
}
```

###**拣货/picking**(尤为复杂)
* 本来准备用两个Fragment分别来区分MY和SG的条件筛选，但是....,所以MY和SG共用的是一个UI，就会出现各种Ui控件上的显示和隐藏（请轻喷）；

* 此处使用的MVP的模式开发的，除了adapter中的Save 以及 点击用户名更改标记颜色的接口是杂糅在Adapter其他请求数据的接口都是在Model中，
处理Ui的方法都是在View中，逻辑在Presenter中（请轻喷+1）；

*注意细节问题

#### SG

##### 进入界面就会调用接口  

 ```
    list<TDeliveryMethod> UserGetDeliveryMethod(),
    //返回值的
    struct TDeliveryMethod{
    1:required string deliveryCode;
    2:required string deliveryName;     //Home Delivery MRT Collection Self_Collection Neighborhood Station 
    3:required list<TCollectionStation> collectionStations;     //deliveryMethod对应的详细StationName，只有当你选择self-station 用到了这个List，其他的选项都是单独请求的接口获得的list；
}
 ```

##### 不同DeliveryMethod对应的Ui文字的改变

```java
 if (CODE_HOME.equalsIgnoreCase(currentDeliveryCode)) {
                    btnDeliveryMen.setHint(R.string.all_driver);
                    btnPickupTime.setVisibility(View.VISIBLE);
                    btnPickupTime.setHint(R.string.all_period);
                } else if (CODE_SUBWAY.equalsIgnoreCase(currentDeliveryCode)) {
                    btnDeliveryMen.setHint(R.string.all_mrt_station);
                    btnPickupTime.setVisibility(View.INVISIBLE);
                } else if (CODE_SELF_COLLECTION.equalsIgnoreCase(currentDeliveryCode)) {
                    btnDeliveryMen.setHint(R.string.warehouse);
                    btnPickupTime.setVisibility(View.VISIBLE);
                    btnPickupTime.setHint(R.string.all_period);
                } else if (CODE_NEIGHBOURHOOD_STATION.equalsIgnoreCase(currentDeliveryCode)) {
                    btnDeliveryMen.setHint(R.string.neighborhood);
                    btnPickupTime.setVisibility(View.VISIBLE);
                    btnPickupTime.setHint(R.string.All_Station);
                } else {
                    btnDeliveryMen.setHint(R.string.All_Station);
                    btnPickupTime.setVisibility(View.VISIBLE);
                    btnPickupTime.setHint(R.string.all_period);
                }
```

##### 搜索

 - 调用接口

```
     //搜索列表 pick ->search；搜索shipment查询对应的Parcel List

    //SG逻辑当Method是Home时候driver和period,当Method为subway时候period为“”，当Method为selfCollection时候传station和period，当Method为Neighborhood时候传region和Station;

    TFindSubPackageResult UserFindPackageNumbers(1:string localDeliveryMethod,2:string deliveryFromDate,3:string deliveryToDate,4:string stationName,5:string periodName,6:bool isProcessing,7:bool isPick,8:i32 house),
```

**注意：**

当deliveryMehod 为Home Delivery（Home）的时候 ，stationName 对应传入的是driver（司机名） ，periodName 对应传入的就是period（时间段）；

当deliveryMehod 为MRT Collection（subway）的时候 ，stationName 对应传入的是MrtStation ，periodName 对应传入的就是""（空）；

当deliveryMehod 为Self-Collection（selfCollection）的时候 ，stationName 对应传入的是station（自取点名） ，periodName 对应传入的就是period（时间段）；

当deliveryMehod 为Neighborhood Station（Neighborhood）的时候 ，stationName 对应传入的是region（邻里区域） ，periodName 对应传入的就是station（邻里站点远原来的时间段按钮）；

- 刷新数据

    更改BP总数btnBPTotal；

    更新数据` listAdapter.search(subPackages);`

```java
 @Override
    public void searchClicked(TFindSubPackageResult response) {
        btnBPNo.setText(response.totalBoxCount + "B" + response.totalPackageCount + "P");
        subPackages = response.subPackages;
        listAdapter.search(subPackages);
        listAdapter.setCurrentDeliveryCode(btnDeliveryMen.getText().toString());
        checkAll.setChecked(false);
    }
```

- 返回值对应的Item参数

```java
public class TFindSubPackageResult extends BaseModule<TFindSubPackageResult> implements Serializable {
    public int totalPackageCount;
    public int totalBoxCount;
    public java.util.ArrayList<TSubPackage> subPackages;
}
```

totalPackageCout 和totalBoxCount 是总BP设置在筛选条件的XBXP中；

subpackages 对应的是列表的参数和设置；
```java
public class TSubPackage extends BaseModule<TSubPackage> implements Serializable {
    public String nickName;  //UserId
    public String parcelNum; //ParcelNumber 包裹号
    public String shelfNum; //shelfNum 货架号
    public double Weight; //重量
    public int packageCount; 
    public int boxCount;//和包裹号形成0B1P 
    public java.util.ArrayList<String> boxNums;  //打印的箱号，点击包裹号显示的dialog列出
    public boolean isLocked; //是否锁定，true：userid红色，false：黑色
    public int customerId;
    public int shipmentId;
    public int packageId;
    public int deliveryHouse;
    public String packageScanLabelColor;//userId 颜色
    public boolean isModifyed;//改Item颜色为浅绿色
    public boolean isHandCreated;//改item颜色为橙色
    public String remark;//显示remark信息，并且可以跳转到打印信息页面
    public String boxNum;//箱号
}
```

##### 锁定与解锁

- 将勾选中的Item的ShipmentId 组成一个List提供给服务器，以更改包裹状态为锁定和非锁定状态，当包裹为锁定状态，ParcelNum的颜色改为红色，当包裹状态为

- 接口调用
```
    //Pick Form -> Lock 更新ShipmentIsLocked 为true 或者 false;

    //设置shipment状态lock unlock （目前只支持Singapore）

     string UserSetShipmentStatus(1: list<i32> shipmentIds,2: bool isLocked),
```

- 在Presenter中对应的方法是 `  presenter.isLock(getCheckedShipmentIds(), true);`当服务器返回解锁/锁定成功的时候会通过IPickView 中的方法

```java
 @Override
    public void lockSucceeded(boolean isLock, ArrayList<Integer> Shipments) {
        if (isLock) {
            Toast.makeText(PickingActivity.this, R.string.lock_succeeded, Toast.LENGTH_SHORT).show();
        } else {
            Toast.makeText(PickingActivity.this, R.string.unlock_succeeded, Toast.LENGTH_SHORT).show();
        }
        listAdapter.changeLocked(isLock, Shipments);
    }
```

- 刷新列表

接口请求成功后 通过`listAdapter.changeLocked(isLock, Shipments);`这个方法刷新数据，锁定改parcelNumber为红色，未锁定的改ParcelNumber为黑色；

###### checkBox Pick

如果选中pick

列表中不能点击，不显示BP，不显示Save，不能选中；

```java
 if (isPicked) {
                check.setChecked(false);
                check.setVisibility(View.INVISIBLE);
                tvBP.setVisibility(View.INVISIBLE);
                tvSave.setVisibility(View.INVISIBLE);
                tvUserId.setClickable(false);
                tvParcelNo.setClickable(false);
                tvBPNo.setClickable(false);
                tvShelfNo.setClickable(false);
            } else {
                check.setVisibility(View.VISIBLE);
                tvBP.setVisibility(View.VISIBLE);
                tvSave.setVisibility(View.VISIBLE);
                tvUserId.setClickable(true);
                tvParcelNo.setClickable(true);
                tvBPNo.setClickable(true);
                tvShelfNo.setClickable(true);
            }
```

##### 关于所有的All 

     列表中的All都是客户端本地添加的；
```java
 /**
     * 在list中增加All 列表选项；
     *
     * @param list
     * @param str
     * @return
     */

    protected ArrayList<String> addAllInList(ArrayList<String> list, String str) {
        list.add(0, str);
        return list;
    }

```

##### Item操作

###### BP值操作

默认ShelfNum (货架号) 携带BP，如果货架号不为“ ”，包含B字母的就为B,其他的为P，货架号为“ ”，没有BP值；

```java
            //如果这个货架号的字符包含B，就判断这个二级包裹号是B，如果不是，就判断为P，在扫描装箱的提交的时候自动保存。
                if (!TextUtils.isEmpty(subParcel.shelfNum)) {
                    String bp = "";
                    if (subParcel.shelfNum.contains("B")) {
                        bp = "B";
                    } else {
                        bp = "P";
                    }
                }

```

手动更改BP值，通过点击Item的BP数量可以弹出Dialog选择BP，当手动选择了BP，这一个Item就会下沉到最后一行；

```java
  private void changeOrder(TSubPackage subParcel) {
        if (list == null) return;
        if (subParcel != null) {
            list.remove(subParcel);
            list.add(subParcel);
        }
    }
```

###### BP值的保存方式

BP值都是通过map保存

key：ParcelNum

value："B"或者"P"或者" "

```java
 private HashMap<String, String> BPMap = new HashMap<>();//用来记录每一个Item的BP状态；key为ParcelNo；
```

###### SAVE 保存 --- 这个没有在MVP中，直接在Adapter中调用api；

* 当点击Save触发，先判断是否有BP值和boxNum，然后调用借口

```thrift

    //pick->save 【WatSavePickPackageAction】
    //保存BP值到PackagerLocalDeliveryLocation.Box、Bag；
    //subPackageNum:二级包裹号
    //BOrP:"B"，"P"; "B"d对应的是BoxCount，”p“对应的是pacelCount
    //isForceSave:是否强制保存；
    //返回值：如果保存成功返回”“;保存失败返回失败信息字符串；
    string UserSavePickSubPackage(1: string subPackageNum,2: string BOrP,3: string boxNum,4: i32 packageId,5: i32 shipmentId,6: bool isForceSave),
```

* response 保存返回值

当respon为“ ”时候toast 保存成功，不为“ ”弹出错误信息，并且提示是否强制保存！！
```java
private void showSaveFailMsg(String response) {
            AlertDialog.Builder builder = new AlertDialog.Builder(mContext);
            builder.setMessage(response).setPositiveButton(R.string.force_save, new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    save(true);//强制保存
                }
            })
                    .setNegativeButton(android.R.string.cancel, new DialogInterface.OnClickListener() {
                        @Override
                        public void onClick(DialogInterface dialog, int which) {
                            dialog.dismiss();
                        }
                    });
            builder.show();
        }
```


##### 点击包裹号

点击包裹号回弹出打印的包裹号。

##### 关于BP值以及箱号的保存在Map里mian

将BP，check，lock，boxNo 都存到map中对应起来

```java
    private HashMap<String, Boolean> checkMap = new HashMap<>();//用来记录每一个Item的checked状态；key为ParcelNo；
    private HashMap<String, String> boxNoMap = new HashMap<>();//用来记录每一个Item的boxNo，以及color；key为ParcelNo；
    private HashMap<String, String> BPMap = new HashMap<>();//用来记录每一个Item的BP状态；key为ParcelNo；
    private HashMap<Integer, Boolean> lockMap = new HashMap<>();//用来记录每一个Item的BP状态；key为ShipmentId；
```

##### 排序（点击蓝色账号 和 货架号）会进行增序排序

##### 扫描

```java
@Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == ScanParcel2BoxActivity.REQUEST_CODE && resultCode == RESULT_OK && data != null) {
            String box = data.getStringExtra("box");
            ArrayList<String> parcels = data.getStringArrayListExtra("parcels");
            Log.i("box", parcels.toString());
            if (!TextUtils.isEmpty(box) && parcels.size() > 0) {
                saveParcels(box, parcels);
                listAdapter.scanResult(box, parcels);
            }
        }
    }
```

扫描得到一个boxNum 和 一个List<String> parcels,再讲操作的Map中把BP boxnum shipmentId取出来；

```java
 /**
     * 上传所有扫描数据并且上传保存Parcel 到 box
     *
     * @param box
     * @param parcels
     */
    private void saveParcels(String box, ArrayList<String> parcels) {
        //上传保存扫描的box 和Parcel
        Map<String, String> bpMap = listAdapter.getBPMap();
        ArrayList<TSaveSubPkgInfo> pkgInfos = new ArrayList<>();
        for (String parcel : parcels) {
            TSaveSubPkgInfo tSaveSubPkgInfo = new TSaveSubPkgInfo();
            //如果这个货架号的字符包含B，就判断这个二级包裹号是B，如果不是，就判断为P，在扫描装箱的提交的时候自动保存。
            //先从bpMap里面找，没有就从box标记里面取
            String bp;
            if (bpMap.containsKey(parcel)) {
                bp = bpMap.get(parcel);
            } else {
                bp = "";
            }
            tSaveSubPkgInfo.BP = bp;
            tSaveSubPkgInfo.pkgId = getPkgIdBySubPkg(parcel);
            tSaveSubPkgInfo.shipmentId = getShipmentIdBySubPkgNo(parcel);
            tSaveSubPkgInfo.subPkgNum = parcel;
            pkgInfos.add(tSaveSubPkgInfo);
        }
        presenter.saveSubPkgs(box, pkgInfos);
    }
```

调用接口

```java
//pick->save 【WatSavePickPackageAction】：可以保存多个，如果全部保存成功就返回true，如果有不成功的就返回false并且返回错误信息列表；
    //pkgInfos：二级包裹号的信息
    //boxNum: 箱号
    //返回值：如果保存成功返回”“;保存失败返回失败信息；
    TSaveResult UserSavePickSubPackages(1:list<TSaveSubPkgInfo> pkgInfos,2: string boxNum),
```

如果失败了显示失败信息：

```java
/**
     * 扫描之后保存完毕信息的处理；
     * 如果全部保存成功，列表中 remove 保存的数据
     * 如果有保存失败的就Dialog失败的信息，list展示失败的信息；
     *
     * @param result saveResult
     */
    @Override
    public void showSavedMsg(TSaveResult result) {
        if (result.isSucceeded) {
            Toast.makeText(this, R.string.all_are_saved, Toast.LENGTH_SHORT).show();
        } else {
            ArrayList<TSaveResultMsg> msgs = result.msgs;
            showSaveFailedDialog(msgs);
        }
        search();
    }
```



2. MY

目前操作和SG 一样，比较大的区别是

还有麦当劳才能选时间；
```java
                /**
                 * 不是McDonald MY不能点击时间，在MY，只有McDonald才能有送货时间；
                 */
                if (CountryInfo.isMalaysia() && !btnDeliveryMen.getText().toString().contains(MCDONALD_STRING)) {
                    btnTimeStart.setText("");
                    btnTimeEnd.setClickable(false);
                    btnTimeEnd.setText("");
                    btnTimeStart.setClickable(false);
                } else {
                    timeCanClick();
                }
                if (currentDeliveryCode.equalsIgnoreCase("McDonald") && CountryInfo.isMalaysia()) {
                    btnTimeStart.setClickable(true);
                    btnTimeEnd.setClickable(true);
                }
```

### 配送/out for delivery

### 自取上架/ready for collection

### 搜索包裹/search parcel

### 包裹列表/parcel list

### 加盟店/partner shop

* 这里只是一个WebView 注入了Cookie 调用了Url
```java
 public static String getPartnerShopUrl() {
        return PARTNER_SHOP_URL = !isLiving ? "http://ezdelivery.65emall.net/" : "http://pdt.65daigou.com/commisions/index.html";//区分Living和Testing
    }
```

### 设置/my setting

* 更改密码

* 显示用户名

* VersionCode
