## 诗三百·人工智能在线诗歌写作平台开发教程

一步步教你搭建人工智能写诗平台，支持AI作诗，藏头诗生成，AI填词，自动对联，目录如下：
* 数据集预处理
* 写诗模型搭建
* BeamSearch奖惩机制
* Flask后端发布
* Vue前端开发
* 高并发架构优化
* 敏感词过滤
* 运营推广SEO优化

### 数据集预处理

数据集采用了[Werneror收集的85万古诗词](https://github.com/Werneror/Poetry)， [王斌收集的70万首对联](https://github.com/wb14123/couplet-dataset) ，[sheepzh收集的8万首现代诗](https://github.com/sheepzh/poetry) 预处理格式如下：

需要自己写个程序判断一下古诗词输入什么格律或词牌, 并转换成如下格式保存到csv文件

输入： 题目&&格律    
输出： 对应的古诗

格律详情如下：  
古诗： 五言绝句，七言绝句，五言律诗，七言律诗  
藏头诗： 五言绝句_藏头诗，七言绝句_藏头诗, 五言律诗_藏头诗， 七言律诗_藏头诗  
词牌： 浣溪沙、鹧鸪天、蝶恋花、西江月、清平乐、菩萨銮、点绛唇、念奴娇、满江红、浪淘沙等  
对联： 上联  

#### 例子：
输入：相思&&五言绝句  
输出：红豆生南国，春来发几枝。愿君多采撷，此物最相思。

输入：春望&&五言律诗  
输出：国破山河在，城春草木深。感时花溅泪，恨别鸟惊心。烽火连三月，家书抵万金。白头搔更短，浑欲不胜簪。

输入：早发白帝城&&七言绝句  
输出：朝辞白帝彩云间，千里江陵一日还。两岸猿声啼不住，轻舟已过万重山。

输入：锦瑟&&七言律诗  
输出：锦瑟无端五十弦，一弦一柱思华年。庄生晓梦迷蝴蝶，望帝春心托杜鹃。沧海月明珠有泪，蓝田日暖玉生烟。此情可待成追忆，只是当时已惘然。

输入：朝千两轻&&七言绝句_藏头诗  
输出：朝辞白帝彩云间，千里江陵一日还。两岸猿声啼不住，轻舟已过万重山。

输入：一曲新词酒一杯&&浣溪沙  
输出：一曲新词酒一杯，去年天气旧亭台。夕阳西下几时回？无可奈何花落去，似曾相识燕归来。小园香径独徘徊。

输入：一觉睡西天，谁知梦里乾坤大&&上联  
输出：只身眠净土，只道其中日月长

输入：只身眠净土，只道其中日月长&&上联  
输出：一觉睡西天，谁知梦里乾坤大

经过预处理，大概统计了一下，古诗有70多万首，对应的藏头诗也有70多万首，藏头诗对应的主题不一定取所有字，可以随机取前几个字，词有3万多首，对联有70万副，上下联对调可以扩增到140万副。由于词牌比较少，需要做数据增强，我用tfidf + textrank算法对每首词取top10个主题词作为输入，对应的词内容还是不变，这样扩增到30多万首左右，总共310万的数据集。

### 写诗模型搭建
写诗模型采用苏剑林老师的[bert4keras](https://github.com/bojone/bert4keras/tree/master/)的[seq2seq](https://github.com/bojone/bert4keras/blob/master/examples/task_seq2seq_autotitle.py), 采用[RoBerta_wwm_ext](https://github.com/ymcui/Chinese-BERT-wwm)预训练模型进行训练，对应好古诗数据集的输入输出就可以训练模型了。

每训练一个epoch，可以在训练集里随机找三个样本进行测试，评估效果，参考代码如下：
```
class Evaluate(Callback):
  def __init__(self,model, type):
      self.lowest = 1e10
      self.model = model
      self.type = type

  def on_epoch_end(self, epoch, logs=None):
    idxs = np.random.choice(range(len(newsdata)), 3)
    for idx in idxs:
      if self.type == 'coupletpoem':
        cptype = newsdata['input'].iloc[idx].split('&&')[1]
        print('格式：', cptype if cptype !='上联' else '对联')
        print('输入：', newsdata['input'].iloc[idx].split('&&')[0])
        try:
          if cptype == '上联':
            res = couplet_gen_sent(self.model, newsdata['input'].iloc[idx][:maxlen])
          elif cptype in ['五言绝句','七言绝句','五言律诗','七言律诗','五言绝句_藏头诗','七言绝句_藏头诗','五言律诗_藏头诗','七言律诗_藏头诗']:
            res = poem_gen_sent(self.model, newsdata['input'].iloc[idx][:maxlen])
          else:
            res = ci_gen_sent(self.model, newsdata['input'].iloc[idx][:maxlen])
          print('输出：', newsdata['output'].iloc[idx])
          print('预测：', res)
        except Exception as e:
          traceback.print_exc()
          print(e)
```

RTX2080Ti下大概训练一天左右，loss降到3.5以下，模型已经基本上能学会写诗了，包含不同格律诗的长度及平仄押韵，但不保证，再训练几天，loss降到2.6以下，写得诗词符合格律的概率就大很多了。

### BeamSearch 奖惩激励
虽然模型基本上学到了古诗的平仄押韵格律，生成的大部分诗都能符合格律，但是不能保证生成的所有诗词都满足格律，所以需要在BeamSearch的时候引入奖惩激励。由于古诗，词牌，对联的格律规则不一样，需要分开写：

#### 古诗规则：
- 古诗里面尽量不出现重复的字，如果出现重复的字，在候选集里面惩罚，但是允许律诗的颔联和颈联出现连续两个字的叠词，如无边落木萧萧下，不仅长江滚滚来；
- 绝句的第二句和第四句最后一个字押韵，律诗每一联的最后一个字押韵。在古诗规则中，一般押韵的字无论采用平水韵还是中华新韵，都是平声，所以第一个位置押韵的字，只要是韵表里的字都奖励，后面押韵的字只对和第一个押韵字同韵的字才奖励；
- 由于生成的古诗第一个字和输入题目第一个字有很大概率会重复，这里引入了惩罚，但是不包括藏头诗  

下面还是基于苏老师老版本的beamsearch代码修改的，供参考

```
poemyun = {
  'a': '啊腌扒叭巴吧芭岜疤笆粑豝嚓跨叉杈差咖瓜抓胍榻喇哈阿花哗加茄迦痂枷耞珈袈嘉佳家傢葭豭咖夸姱啦妈摩嬷趴凹咱葩杉沙莎痧鲨纱砂他她它哇洼蛙娲虾丫呀鸦哑桠查楂喳呱欻旮笳拉垃吗蚂仨裟砂趿渣揸馇挝阿八捌擦插锸耷哒嗒鎝褡发筏夹嘎刮栝鸹拉邋抹掐袷葜撒杀刹铩煞刷趿塌溻禢踏挖呷瞎鸭压押扎匝咂拶吒哳浃撒答啊茶查搽嵖猹槎楂碴苴垞蛤华哗骅铧麻嘛蟆拿扒杷爬钯耙筢琶娃霞遐瑕暇牙伢芽岈玡蚜崖涯睚衙拔茇菝跋魃察茬檫达鞑沓怛妲笪靼答跶乏伐茷垡砝阀罚嘎滑划猾夹浃郏荚铗蛱恝戛颊旯拉匣狎挟柙侠峡狭硖辖黠杂砸扎札轧闸铡喋剳',
  'o': '波播菠玻嶓搓磋蹉瑳多哆呙锅卜埚涡坡颇摸阿陂莎唆娑梭挲疙睃圪嗦嗍鲅蓑拖莴倭唷涡窝蜗踒阿婀痾哥歌戈呵科蝌柯疴苛珂窠轲颗屙菏棵髁的了么呢车奢赊畲遮仡猞拨鱿趵钵饽剥逴踔戳撮咄剟掇裰郭崞聒蝈豁劐擢捋泊泼钹说缩托脱喔拙捉苛桌倬涿焯作嘬鸽割搁喝磕瞌榼脖嵯痤瘥鹾罗萝啰逻脶猡锣椤箩骡螺谟馍馍摹模麽摩磨嬷蘑磨魔挪娜傩婆鄱繁皤驮佗陀坨驼柁砣鸵酡跎蛇鼍鹅蛾娥莪俄峨哦讹和禾何河荷阇膜婆皤沱渮哪挼孛荸伯驳帛瓝泊柏勃钹铂舶博鹁浡渤搏箔膊踣薄馞欂襮礴夺度铎踱怫佛掇咄裰剟国掴帼漍腘虢馘活嗑橐灼彴茁踔卓斫浊酌浞诼着啄琢椓蠋擢濯镯昨作笮阁葛蛤颌合涸盒膜拙棁捽貉曷盍阖壳德得额鹖则舴晢蜇革格鬲隔嗝槅膈滆塥鎘骼纥劾阂核翮壳咳颏舌则责择咋泽啧帻舴箦赜折哲辄蛰谪摺磔辙翟',
  'ie': '爹阶皆喈嗟街湝乜咩些靴耶倻椰偕掖瘪憋鳖跌颏疙节截疖角圪结接秸揭噘撅捏撇瞥切缺阙贴得怗贴帖楔歇蝎削薛噎曰约瘸斜邪偕谐鞋携爷耶茄伽鲑踅椰别蹩迭垤昳絰瓞谍堞耋揲喋牒叠碟蝶艓蹀孑节讦劫劼杰诘拮洁结桔桀捷偈玦觉决绝倔桷掘崛脚觖厥劂谲獗蕨橛噱爵蹶矍嚼爝攫钁协胁挟絜颉撷勰襭穴学噱',
  'ai': '开哎哀埃挨娭唉欸掰偲钗差揣呆该陔垓荄赅乖揩腮毸鳃筛酾衰摔苔台胎歪灾哉栽甾斋拍摘拆塞猜挨皑癌才财材裁侪柴豺还孩骸徊怀淮槐踝来莱崃徕涞埋霾俳排徘牌簰邰苔抬骀炱鲐白宅翟',
  'ei': '微欸陂杯卑背悲碑鹎衰崔催摧縗吹榱炊堆飞妃非菲啡騑绯馡蜚扉霏鲱归圭龟妫规邽扳闺硅黑嘿傀瘣瑰鲑灰诙虺挥咴恢袆珲豗晖辉翬麾徽亏刲岿勒悝盔窥胚呸绥衃醅虽荽睢濉忒推危威逶偎隈葳椳煨溦巍蝛薇追骓锥椎欸垂陲捶椎槌棰倕锤箽肥淝腓回茴洄蛔奎逵馗隗葵揆骙暌魁戣睽蝰累雷嫘缧擂檑礌镭羸罍虆玫枚眉莓脢梅嵋猸湄邳媒楣煤酶镅霉糜陪培赔没裵蕤绥隋随遂谁颓韦为圩贼违围帏沩桅唯帷惟薇维嵬巍潍闱',
  'ao': '凹熬包薄苞胞孢剥龅煲褒标飚彪骠镖瘭飙操糙藨镳瀌膘杓猋骉摽操抄怊钞超剿潦刀叨忉刁汈蛁雕貂叼碉凋鲷高皋羔槔睾膏篙糕蒿薅嚆交姣骄艽郊茭浇娇胶椒蛟焦蕉教跤僬鲛嶕礁噍鹪轇尻捞撩猫喵孬抛脬泡漂剽慓飘缥螵僄悄磽磽锹劁敲橇缲搔骚挑缫臊捎烧梢稍筲艄蛸叨涛绦掏滔韬弢饕慆佻祧肖枭枵哓骁嘐逍虓鸮消宵绡萧硝销削蠨蛸翛箫潇霄魈歊嚣幺约夭吆妖要喓腰邀遭糟钊招昭嘲啁着朝约剥削豪敖璈遨嗷廒獒熬隞嶅聱翱鳌鏖螯骜薄雹曹槽螬漕嘈晃巢朝嘲潮捡捯号嗥毫壕濠嚎蠔嚼劳崂痨牢捞唠醪聊辽疗僚漻寥嘹獠寮缭嫽燎憭髎毛矛茅牦旄酕锚髦蝥蟊苗描瞄饶挠饶蛲猱呶刨咆狍庖炮袍匏嫖朴瓢薸乔侨荞峤桥硚翘谯荍鞽憔樵瞧荛桡蛲饶娆桡蛲饶娆苕韶勺芍咷梼逃洮桃陶萄梼啕淘綯醄鼗条岹苕调笤齠蜩迢髫岧鲦崤淆洨爻尧肴淆轺峣陶姚窑谣摇徭遥猺瑶飖鳐轺凿着',
  'ou': '抽掐紬瘳丢都兜蔸勾佝沟枸钩缑篝巘鞲齁勼纠鸠究赳阄湫揪啾蝤轇芤抠眍溜熘搂喽哞妞区讴沤瓯欧殴呕鸥丘邱龟秋蚯湫楸鹙鳅鞦收搜嗖锼馊廋溲飕艘偷修脩休咻羞鸺貅馐髹优攸忧悠呦幽麀舟州谄侜周洲粥啁赒輈诪邹郰緅驺诹陬鲰粥儔帱畴筹酬俦踌惆绸稠禂仇愁雠侯喉猴篌瘊糇流留榴骝刘浏瘤琉硫旒鹠遛镏飗瘤鎏娄楼偻蒌喽耧蝼髅牟眸谋蛑缪鍪牛抔掊裒囚仇犰求毬虬泅俅璆酋逑球遒赇裘璆蝤柔揉糅煣蹂鞣头投骰尤犹疣鱿莸铀由邮油游猷繇蝣妯轴',
  'an': '安氨唵桉庵谙鹌鞍盦扳班颁斑攽般搬瘢皤癍参骖餐觇搀幨襜川穿氽撺镩丹担单眈酖耽郸聃禅儋殚箪端帆番蕃幡藩翻干甘杆玕肝柑竿疳尴关观纶官冠矜倌棺瘝鳏顸酣憨鼾欢讙獾驩刊看勘龛堪戡宽髋颟囡番潘攀三叁山芟杉删衫姗珊栅舢扇跚煽潸膻闩拴栓酸坍贪摊滩瘫湍弯剜湾蜿豌糌簪占沾毡旃粘詹谵邅瞻专砖颛钻躜边砭萹笾编煸蝙鯿箯鞭参骖餐掂傎癫滇颠巅戋尖奸歼坚间肩艰监兼菅笺渐犍湔缄蒹煎缣鹣搛熸鞬鳒韉捐涓娟朘圈鹃镌蠲拈蔫片扁偏篇犏翩千仟阡芊扦迁佥钎牵铅悭谦签愆鹐骞搴磏諐褰圈悛棬弮天添黇仙先纤氙忺籼掀铦酰跹锨鲜暹骞轩宣谖萱揎喧瑄煖禤暄儇懁咽恹殷胭烟焉崦阉阏奄淹腌湮鄢嫣燕鸢眢鸳冤渊鹓箢寒残蚕惭单鋋馋谗婵禅孱缠蝉廛僝潺澶镡蟾镵巉躔传船遄椽攒凡矾烦墦蕃攀樊璠燔繁蘩邗汗邯含函琀焓晗涵韩还环桓圜阛寰缳鬟郇荁洹貆澴轘兰岚斓拦栏婪阑蓝谰澜褴篮斕镧峦娈孪挛鸾脔滦栾銮蛮谩蔓馒瞒鞔鳗鬘男南难喃楠爿胖般盘磻磐蹒蟠蚺然燃髯坛昙倓郯谈弹覃谭痰潭檀团漙抟咱丸纨完玩顽刓汍烷奁连镰怜帘莲涟联裢鲢廉濂鐮鬑磏眠绵棉年粘黏鲇便骈胼蹁钤前虔钱钳乾掮潜黔犍权全佺诠荃泉辁拳铨痊惓筌蜷醛鳈鬈颧田佃畋恬钿甜湉填阗闲贤弦咸挦涎娴衔舷痫鹇嫌玄悬旋漩璇延蜒严言芫妍岩炎沿铅研盐阎筵颜檐元园员沅垣湲袁原圆鼋援媛缘猿塬嫄源羱辕橼',
  'en': '奔贲锛玢宾彬傧斌滨缤槟濒豳参抻郴伧琛嗔汶瞋春椿蝽村皴踆惇吨墩礅敦蹲恩分芬吩纷玢菌氛棻雰根跟昏阍惛婚巾斤今紷金津衿矜筋禁襟军均龟君钧皲坤昆崑裈堃焜琨髡鹍锟鲲抡拎闷喷拼姘钦侵亲衾駸嵚囷逡森申伸身呻砷侁诜参绅珅莘娠深糁燊孙荪狲飧吞暾温瘟心芯骎辛忻昕欣炘锌新歆薪馨鑫勋埙熏薰薫獯曛醺窨因阴茵洇裀荫音姻氤殷堙喑闉愔禋晕缊氲煴赟贞针侦浈珍帧胗真桢砧祯蓁斟甄獉溱榛箴臻迍肫窀谆尊遵樽鳟岑涔臣尘辰沉忱陈宸晨谌纯莼唇淳鹑漘醇存蹲坟汾棼焚濆痕贲贲浑珲馄混哏魂邻林临淋琳粼磷潾嶙遴霖辚瞵鳞麟仑伦论抡沦纶轮门扪们民忞旻岷缗您盆湓贫频嫔颦芹芩矜秦琴覃禽勤懃擒噙螓裙群人壬仁任神什屯囤饨豚臀文纹炆闻蚊雯旬郇寻巡询洵荀荨峋恂鲟循吟垠龈狺訚崟银荧淫寅蟫鄞夤嚚霪云匀芸员沄纭昀畇筠耘筼麇',
  'ang': '肮邦帮梆浜仓苍沧鸧舱昌倡菖猖阊娼伥创疮窗胯当珰铛裆筜方坊芳枋邡钫冈岗刚矼肛纲钢缸釭罡堽光咣胱夯荒肓塃慌江将姜豇浆僵螀缰疆康慷糠囊匡劻诓恇筐牤乓雱滂膀枪锵羌戗戕将腔蜣铰镪嚷丧桑伤汤殇商觞墒熵双泷霜孀孀骦鹴汤铴耥嘡镗蹚汪乡芗相香厢湘缃箱襄骧镶秧央湍殃鸯鞅赃脏臧张章獐彰嫜璋樟蟑妆庄桩装卬昂藏长场苌肠尝常偿徜裳嫦床幢防妨坊忍肪鲂房行吭迒杭絎航颃皇黄凰隍喤煌遑徨湟惶粕锽潢璜蝗篁磺蟥簧鳇扛狂诳鵟郞狼郎琅榔桹琅廊嫏樃硠锒稂鎯螂良俍莨凉梁椋量粮粱邙芒忙杧盲氓茫硭铓牻嚢馕娘彷庞逄旁蒡膀磅螃强墙蔷嫱樯蘘灢禳瓤唐堂棠塘搪糖溏瑭樘膛赯螗螳鄌亡王详降绛庠祥翔扬阳羊玚飏炀杨旸佯疡徉洋',
  'eng': '庚更横盛正应乘胜兴行刑莹茔宁凭暝钉称令并伻崩听祊绷嘣冰兵槟屏柽琤称蛏铛赪撑噌瞠灯登噔蹬镫丁仃叮玎盯钉疔酊靪丰风封枫疯峰烽葑锋蜂更庚耕赓鹒羹亨哼精茎惊京经睛泾荆菁旌晶粳兢鲸坑吭硁铿蒙抨怦砰烹嘭乒俜娉青轻氢倾卿圊清蜻鲭扔僧升生声牲笙甥鼪厅汀听翁嗡兴星狚惺腥应英莺婴撄嘤罂缨璎樱鹦媖瑛膺鹰曾增憎缯罾正争筝蒸征怔挣峥狰钲症烝睁铮稳东冲憧充忡翀舂惮艟匆苁囱枞葱骢璁聪熜冬咚鸫工弓公功攻供肱宫恭蚣躬龚觥哼轰哄訇烘薨埛駉扃空倥崆屄箜忪松凇菘嵩恫通嗵瘑凶兄芎匈汹恟胸佣痈拥邕鄘雍墉慵庸镛壅臃鳙中忪忠终钟盅衷螽宗综棕踪鬃层曾嶒成酲丞呈枨诚承城宬乘盛程惩裎塍澄橙冯逢缝恒姮桁珩横衡蘅楞棱伶灵苓蛉囹泠玲令瓴铃鸰凌陵聆菱棂暹舲翎羚绫棱零龄鲮酃氓虻萌蒙盟甍瞢幪濛曚矇朦艨檬名茗明鸣冥铭洺蓂溟暝瞑螟能棕拧咛狞柠凝芃仍朋膨堋澎彭棚蓬硼鹏篷鬅平冯评坪苹凭枰洴帡屏瓶萍勍情晴檠擎黥绳渑疼腾誊螣藤廷亭停庭蜓婷霆行形邢陉型荥盈萤莹营萦楹滢蝇潆贏赢瀛迎虫重崇从丛尝悰琮弘红吰闳宏泓荭虹闳洪翃鸿黉龙栊茏咙泷珑眬胧昽聋笼隆癃窿农侬哝傢浓脓秾邛穷茕穹藭筇琼蛩跫戎茸荣绒容崂蓉溶瑢榕融嵘同彤侗苘峒桐砼垌佟烔鲷峂樟僮铜童潼瞳朣曈艟雄熊喁颙',
  'di': '氐低羝堤提蹂揉沤几讥叽饥玑机乩肌矶鸡奇屐剞笄姬基期赍犄嵇畸跻箕稽齑丽畿羁咪眯妮丕邳裼批纰坯披砒沏妻栖萋期攲梯蹊欺兮西溪希茜郗稀熙牺唏悕晞傒豨僖嘻奚嬉熹樨羲蹊粞犀曦醯巇鼳伊铱医衣依袆咿猗漪噫繄居车苴拘驹俱狙罝疽据琚趄睢裾区岖佉驱祛蛆躯焌趋黢呈圩盱须虚需嘘墟胥湑谞訏迂纡淤逼嘀滴圾芨唧积击缉激唧襀劈噼霹七柒凄欹戚缉喴漆剔踢夕吸汐昔析矽穸息悉蜥晰淅惜翕蜤锡晰熄噏膝螅歙螅窸蟋一壹揖曲屈掬鞠鋦离蛐欻戌厘狸离骊梨犁鹂喱蓠漓缡璃嫠孷藜黎鲡罹篱黧蠡弥迷眯猕谜糜麋靡蘼醾尼泥坭呢妮輗怩倪霓猊鲵麑皮陂疲枇芘狓毗蚍陴埤啤琵脾裨蜱罴貔鼙齐祈圻芪岐荠祁其奇跂祈祇俟耆颀脐旂萁畦跛崎淇颈骑琪琦棋蛴祺錡綦旗蕲鳍麒鬐荑绨提啼鹈騠缇稊题醍蹄仪圮夷痍匜迤饴怡宜荑贻沂诒眙簃姨胰扅蛇移遗颐椸疑嶷彝儿而驴闾榈劬渠蕖瞿蘧氍癯衢蘧鸲徐于予妤玙余馀欤盂臾鱼竽舁俞谀娱萸雩渔隅揄喁畲逾腴渝愉瑜榆虞愚觎舆窬髃荸鼻锹迪笛的荻敌涤微嫡翟镝及伋吉岌汲级极即舍诘亟革芨急疾棘殛蕺集蒺楫辑嵴踖瘠藉籍习席觋袭媳嶍隰檄熄锡局桔菊焗跼橘曲',
  'i': '哧蚩鸱絺眵笞瓻摛嗤痴媸螭魑呲差疵跐骴尸师诗思鸤坻絁狮葹施訾酾司丝私咝鸶仂斯蛳幼飔厮罳澌撕嘶之卮知支芝吱枝肢栀胝祗脂蜘仔吱孜咨姿兹赀资訾泑嗞缁辎赠粢孳滋趑觜锱龇菑鲻吃失虱湿只织池弛驰迟踟坁持匙漦墀篪词茈茨祠瓷辞慈磁雌慈糍鹚时拾十石实识食蚀仁浉执直侄值职填植殖絷跍摭踯',
  'u': '逋蜅初粗都阇嘟夫肤玞鄜孵敷估姑咕沽孤轱畈鸪罛菇菰蛄辜酤呱觚箍乎呼毂糊刳矻枯骷撸噜铺痡殳书倏姝抒纾枢妷殊梳舒摴樗摅毹输疏蔬苏稣酥栈乌圬邬污呜於钨巫朱洙侏诛茱珠株诸铢猪蛛槠潴橥租菹出督忽胡惚唿淴哭窟仆菩不鹄扑噗叔修辞菽淑窣突秃凸葖屋诬刍除厨锄滁蜍橱篨躇蹰雏徂殂凫扶孚罘苻莩俘浮蚨桴符涪蜉鞭郛狐弧縠和壶葫猢瑚糊蝴糊醐湖瓠鹕卢芦庐垆炉泸栌轳胪鸬颅舻鲈模奴孥驽匍莆脯葡蒲酺如茹儒薷嚅濡孺襦蠕图荼徒途涂菟屠酴无毋芜吾吴唔梧蜈鼯醭毒独读渎椟牍犊髑顿弗佛艴茀幅袱刜拂彿氟绋茀怫芾伏洑茯绂祓服菔芙匐福蝠辐幞襆囫斛觳鹘醭璞濮孰赎塾熟秫俗竹术竺逐烛舳瘃躅足卒崒族镞'
}

poemyuns = [poemyun[key] for key in poemyun]
poemyunsall = ''.join(poemyuns)
poemobj = {
  '五言绝句': [10,22],
  '五言律诗': [10,22,34,46],
  '七言绝句': [14,30],
  '七言律诗': [14,30,46,62]
}

poemobjreword = {
  '五言绝句': [],
  '五言律诗': [11,35],
  '七言绝句': [],
  '七言律诗': [15,47]
}

def poem_gen_sent(self, model, tokenizer, s, topk=2):
    """beam search解码
    每次只保留topk个最优候选结果；如果topk=1，那么就是贪心搜索
    """
    #print('topk: ',topk)
    if '&&' not in s:
      s = s + '&&七言律诗'
      print(s)
    token_ids, segment_ids = tokenizer.encode(s[:maxlen])
    target_ids = [[] for _ in range(topk)]  # 候选答案id
    target_scores = [0] * topk  # 候选答案分数

    poemtype = s.split('&&')[1].split('_')[0]
    #print(poemtype,poemobj[poemtype])

    comma, _ = tokenizer.encode('，。？！、；：')
    comma = comma[1:-1]
    yunlist = {}
    for i in range(maxlen):  # 强制要求输出不超过maxlen字
        _target_ids = [token_ids + t for t in target_ids]
        _segment_ids = [segment_ids + [1] * len(t) for t in target_ids]
        _probas = model.predict([_target_ids, _segment_ids
                                 ])[:, -1, 3:]  # 直接忽略[PAD], [UNK], [CLS]
        _log_probas = np.log(_probas + 1e-6)  # 取对数，方便计算
        _arg = _log_probas.argsort(axis=1)[:, -32:]

        for j in range(topk):
          if i == poemobj[poemtype][1]:
            yunword = tokenizer.decode([target_ids[j][poemobj[poemtype][0]]])
            for item in poemyuns:
              if yunword in item:
                yunlist[j] = item
                break
          for k in _arg[j]:
            # 之前已经出现过引入惩罚
            if k + 3 not in comma:
              if len(poemobjreword[poemtype]) == 2 and i > poemobjreword[poemtype][0] and i < poemobjreword[poemtype][1] and k + 3 in target_ids[j][:-1]:
              # 律诗第二联和第三联可以出现连着两个字
              #print('重复引入惩罚:', k+3, tokenizer.decode([k+3]), _log_probas[j][k], _log_probas[j][k] * 100)
                _log_probas[j][k] = _log_probas[j][k] * 10000
              elif k + 3 in target_ids[j]:
                _log_probas[j][k] = _log_probas[j][k] * 10000
            if i == poemobj[poemtype][0] and tokenizer.decode([k+3]) in poemyunsall:
              # 第一个押韵的字在平水韵表里面奖励
              _log_probas[j][k] = _log_probas[j][k] / 100
            elif i in poemobj[poemtype][1:] and j in yunlist and tokenizer.decode([k+3]) in yunlist[j]:
              # 后面押韵的字要跟第一个押韵的字在同一个韵里面
              _log_probas[j][k] = _log_probas[j][k] / 100

            # 第一个字和标题第一个字重复惩罚
            if i == 0 and tokenizer.decode([k+3]) == s[0] and '藏头诗' not in s:
              _log_probas[j][k] = _log_probas[j][k] * 10000
            elif tokenizer.decode([k+3]) in s and '藏头诗' not in s:
              _log_probas[j][k] = _log_probas[j][k] * 2


        _topk_arg = _log_probas.argsort(axis=1)[:, -topk:]  # 每一项选出topk
        _candidate_ids, _candidate_scores = [], []
        for j, (ids, sco) in enumerate(zip(target_ids, target_scores)):
            # 预测第一个字的时候，输入的topk事实上都是同一个，
            # 所以只需要看第一个，不需要遍历后面的。
            if i == 0 and j > 0:
                continue
            for k in _topk_arg[j]:
                _candidate_ids.append(ids + [k + 3])
                _candidate_scores.append(sco + _log_probas[j][k])
        _topk_arg = np.argsort(_candidate_scores)[-topk:]  # 从中选出新的topk
        target_ids = [_candidate_ids[k] for k in _topk_arg]
        target_scores = [_candidate_scores[k] for k in _topk_arg]
        best_one = np.argmax(target_scores)
        if target_ids[best_one][-1] == 3:
          return tokenizer.decode(target_ids[best_one])
    # 如果max_output_len字都找不到结束符，直接返回
    return tokenizer.decode(target_ids[np.argmax(target_scores)])
```

#### 词牌规则：
- 每个词牌的格律不一样，一个词牌写一个规则太麻烦，我这里就引入了重复字和标题首字惩罚，规则和古诗一样，主要是让模型自己学习各个词牌的格式及韵律。

#### 对联规则：
- 下联和上联的长度一致
- 下联不能出现上联中的字，对beamsearch候选集中上联出现字进行惩罚，但是不包括标点符合
- 上联中如果出现重复的字，下联对应位置进行奖励

```
def couplet_gen_sent(self, model, tokenizer, s, topk=2):
    """beam search解码
    每次只保留topk个最优候选结果；如果topk=1，那么就是贪心搜索
    """
    s = s.replace(' ','，').replace('。','')
    token_ids, segment_ids = tokenizer.encode(s)
    target_ids = [[] for _ in range(topk)]  # 候选答案id
    target_scores = [0] * topk  # 候选答案分数

    comma, _ = tokenizer.encode('，。？！、；：')
    comma = comma[1:-1]
    for i in range(len(s.split('&&')[0])):  # 强制要求输出不超过title_maxlen字
        _target_ids = [token_ids + t for t in target_ids]
        _segment_ids = [segment_ids + [1] * len(t) for t in target_ids]
        _probas = model.predict([_target_ids, _segment_ids
                                 ])[:, -1, 4:]  # 直接忽略[PAD], [UNK], [CLS]
        _log_probas = np.log(_probas + 1e-6)  # 取对数，方便计算
        _arg = _log_probas.argsort(axis=1)[:, -64:]

        for j in range(topk):
          if token_ids[i+1] in token_ids[1:i+1]:
            # 第i个值的输入之前出现过，奖励
            idxs = [ii for ii,x in enumerate(token_ids[1:i+1]) if x==token_ids[i+1]]
            target_idxs = [target_ids[j][ii] for ii in idxs]
          else:
            idxs = []
            target_idxs = []

          for k in _arg[j]:
            if k + 4 not in comma and k + 4 in token_ids:
              # 当前不为标点，如果在输入里面引入惩罚
              _log_probas[j][k] = (_log_probas[j][k]-1) * 1000
            elif k + 4 in comma and token_ids[i+1] not in comma:
              # 当前为标点，但是输入不是，引入惩罚
              _log_probas[j][k] = (_log_probas[j][k]-1) * 1000
            elif k + 4 in comma and token_ids[i+1] in comma and k+4 == token_ids[i+1]:
              # 当前为标点，预测也为标点，奖励
              _log_probas[j][k] = _log_probas[j][k] / 10
            elif k + 4 in target_idxs:
              _log_probas[j][k] = _log_probas[j][k] / 1000
            elif k + 4 in target_ids[j]:
              # 如果第i个值的输入之前没有出现过，惩罚
              _log_probas[j][k] = (_log_probas[j][k]-1) * 1000

        _topk_arg = _log_probas.argsort(axis=1)[:, -topk:]  # 每一项选出topk
        _candidate_ids, _candidate_scores = [], []
        for j, (ids, sco) in enumerate(zip(target_ids, target_scores)):
            # 预测第一个字的时候，输入的topk事实上都是同一个，
            # 所以只需要看第一个，不需要遍历后面的。
            if i == 0 and j > 0:
                continue
            for k in _topk_arg[j]:
                _candidate_ids.append(ids + [k + 4])
                _candidate_scores.append(sco + _log_probas[j][k])

        _arg = np.argsort(_candidate_scores)
        _topk_arg = np.argsort(_candidate_scores)[-topk:]  # 从中选出新的topk
        target_ids = [_candidate_ids[k] for k in _topk_arg]
        target_scores = [_candidate_scores[k] for k in _topk_arg]
    return tokenizer.decode(target_ids[np.argmax(target_scores)])
```

### Flask后端发布
训练模型的时候建议每个epoch保存最优的权重参数到.weight文件，训练结束后可以读取最优的weight权重参数，然后保存h5文件。注意保存h5文件前千万不要对模型进行编译，要不然保存的h5文件会非常大，本来300多M的文件，编译之后会达到1个多G，实际线上预测的时候也不会用到模型编译，所以只要保存网络结构和权重参数到h5就用可以了，大小只会比.weight权重文件稍微大一点。当然你也可以不保存h5文件，直接初始化模型并加载权重参数，但是这样会预加载两次权重参数，第一个是预训练的权重参数，第二个是自己训练的权重参数，比较慢，而且代码也比较长，所有最优建议还是保存到h5文件比较好。

保存之后，就可以把模型初始化和预测代码封装到一个类里面，这样对发布代码也比较比较精简，下面的代码兼容了bert4keras的其他应用，供参考，实际如果只写诗，可以更精简一点。

```
class BertUtil:
  def __init__(self):
    self.modelobj = {}
    self.tokenizerobj = {}
    print('bert util init succeed!')
    
  def init_seq2seq_token(self, premodel, type, level='seq2seq'):
    token_dict = {}
    keep_words = []
    seq2seq_config = seq2seq_config_base %(premodel, level, type)
    config_path, checkpoint_path, dict_path, do_lower_case = getconfig(premodel)

    _token_dict = load_vocab(dict_path)  # 读取词典
    _tokenizer = Tokenizer(_token_dict, do_lower_case=do_lower_case)  # 建立临时分词器

    if type in ['cmrc','infoexts2s','kbqa','mathp']:
      newdict = False
    else:
      newdict = True

    if newdict == True:
      if os.path.exists(seq2seq_config):
        tokens = json.load(open(seq2seq_config))
        print('load config %s succeed!' %(seq2seq_config))
      else:
        exit('invalid config %s' %(seq2seq_config))


      for t in ['[PAD]', '[UNK]', '[CLS]', '[SEP]']:
        token_dict[t] = len(token_dict)
        keep_words.append(_token_dict[t])

      for t in tokens:
        if t in _token_dict and t not in token_dict:
          token_dict[t] = len(token_dict)
          keep_words.append(_token_dict[t])
    else:
      for t in ['[PAD]', '[UNK]', '[CLS]', '[SEP]']:
        token_dict[t] = len(token_dict)
        keep_words.append(_token_dict[t])

      for t, _ in sorted(_token_dict.items(), key=lambda s: s[1]):
        if t not in token_dict:
          if len(t) == 3 and (Tokenizer._is_cjk_character(t[-1])
                              or Tokenizer._is_punctuation(t[-1])):
            continue
          token_dict[t] = len(token_dict)
          keep_words.append(_token_dict[t])

    tokenizer = Tokenizer(token_dict, do_lower_case=do_lower_case)  # 建立分词器
    return tokenizer, keep_words
    
  def init_seq2seq(self, premodel='roberta', type='coupletpoem'):
    level = 'seq2seq'
    modeltype = modeltype_base %(level, type)
    self.tokenizerobj[modeltype], keep_words = self.init_seq2seq_token(premodel, type)
    modelfile = modelfile_h5_base %(premodel, level, type)
    if os.path.exists(modelfile):
      #self.modelobj[modeltype].load_weights(modelfile)
      self.modelobj[modeltype] = load_model(modelfile)
      print('load model %s succeed!' %(modelfile))
    else:
      exit("invalid model %s!" %(modelfile))
      
  def predict_seq2seq(self, type='medicalqa', text='', topk=1, printflag=True):
    modeltype = modeltype_base %('seq2seq', type)
    if printflag:
      print(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()), type, 'topk:', topk)
    if type == 'coupletpoem':
      text = text.replace('##','&&')
      if '&&' not in text:
        text = text + '&&七言律诗'
      cptype = text.split('&&')[1]
      if cptype == '上联':
        text = text.replace(' ','，').replace('。','')
        if topk <= 3:
          result = self.couplet_gen_sent(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        else:
          result = self.couplet_gen_sent_random(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        if printflag:
          print('上联：%s' %(text.split('&&')[0]))
          print('下联：%s' %(result))
      elif cptype in ['五言绝句','七言绝句','五言律诗','七言律诗','五言绝句_藏头诗','七言绝句_藏头诗','五言律诗_藏头诗','七言律诗_藏头诗']:
        if topk <= 3:
          result = self.poem_gen_sent(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        else:
          result = self.poem_gen_sent_random(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        result = result.replace('。','。\n')
        if printflag:
          print(text.split('&&')[0], text.split('&&')[1])
          print(result)
      else:
        if topk <= 3:
          result = self.ci_gen_sent(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        else:
          result = self.ci_gen_sent_random(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
        result = result.replace('。','。\n')
        if printflag:
          print(text.split('&&')[0], text.split('&&')[1])
          print(result)
    else:
      result = self.gen_sent(self.modelobj[modeltype], self.tokenizerobj[modeltype], text, topk=topk)
    return result
```

封装成类之后，就可以通过flask发布了，
```
#!/usr/bin/env python3
#-*- coding:utf-8 -*-

import os,sys
import json
import datetime
import time
#from io import BytesIO
from flask import Flask, request
#from gevent.wsgi import WSGIServer

#from gevent.pywsgi import WSGIServer
#from multiprocessing import cpu_count, Process
#from impala.dbapi import connect
import tensorflow as tf
from keras.backend.tensorflow_backend import set_session

os.environ['CUDA_VISIBLE_DEVICES']='0' # 指定GPU 0
config = tf.ConfigProto()
config.gpu_options.per_process_gpu_memory_fraction = 0.08 
config.gpu_options.allow_growth = True
set_session(tf.Session(config=config))

print('server start...')


from bertutil import BertUtil

#flask app
app = Flask(__name__)

# BERTUTIL
bertutil = BertUtil()
bertutil.init_seq2seq(premodel='roberta',type='coupletpoem')
print('-----------BERT SEQ2SEQ初始化完毕------------')

# seq2seq请求
@app.route("/zqcloudapi/v1.0/nlp/bertseq2seq", methods=['GET','POST'])
def bertseq2seq():
  start =time.time()
  data = request.values['data'] if 'data' in request.values else '虎啸青山抒壮志&&上联'
  type = request.values['type'] if 'type' in request.values else 'coupletpoem'
  topk = int(request.values['topk']) if 'topk' in request.values else 1
  r = bertutil.predict_seq2seq(type=type, text=data, topk=topk)
  end = time.time()
  res={}
  res['result'] = r
  res['timeused'] = int(1000 * (end - start))
  print('使用时间：%sms\n---------' %(res['timeused']))
  return json.dumps(res)

# 跨域支持
def after_request(resp):
    resp.headers['Access-Control-Allow-Origin'] = '*'
    return resp

@app.after_request
def cors(environ):
  environ.headers['Access-Control-Allow-Origin']='*'
  environ.headers['Access-Control-Allow-Method']='*'
  environ.headers['Access-Control-Allow-Headers']='x-requested-with,content-type'
  return environ

def start_web_server(host='0.0.0.0', port=11456):
  print('listen %s:%s' %(host, port))
  app.run(host, port)

if __name__ == "__main__":
  start_web_server()

```

保存文件名为server_poem.py, 用python3 server_poem.py就可以启动了， 服务用0.0.0.0:11456端口监听，如果需要让其他的客户端用ip访问，监听host需要是0.0.0.0，不能是127.0.0.1，否则只能本地访问，可以用netstat查看服务是否启动成功
```
netstat -nlp|grep 11456
```
启动成功后，就可以通过浏览器或客户端脚本访问 http://ip:11456/zqcloudapi/v1.0/nlp/bertseq2seq 并传入参数返回结果

### Vue前端开发
前端最原始的就是html + css + js, 随着技术的发展，出现了bootstrap，vue, react等框架，这里采用Vue框架用于前端开发。前端主要的功能就是给用户提供良好的人机交互界面，把用户输入的主题通过ajax或fetch传给后端，后端计算后并返回给前端展示，这里就不在详细介绍了，有问题的同学可以自己补习了前端相关知识，不是太复杂。

```
fetch(url, {
    headers: {
      'Accept': 'text/plain',
      'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
    },
    method: 'post',
    body: param
  })
    .then(res => {
      return res.text();
    })
    .then(res => {
      callback(res);
    })
    .catch(err => {
      errcallback(err);
    });
```


### 高并发架构优化

上面前后端已经实现了完整的功能，在本地自己部署自己用用已经没有问题了，但是部署到云服务器上，给几百成千甚至上万的用户用，并发瓶颈就出来了，甚至会导致服务器卡死宕机， 下面分别从服务器，模型，后端，接口，前端介绍一下优化的过程：

**1、服务器**  

本项目采用了阿里云服务器，由于写诗模型是用RoBerta seq2seq训练的，权重模型大概有300多兆，使用CPU跑特别慢，在不考虑并发的情况下光写一首诗大概就要好5-9秒，一般用户最大的耐心是3s左右，如果超过3s，网站的体验效果就比较差了，有些用户也不一定有耐心继续往下等，可能就直接关掉了，所以必须使用GPU服务器来跑写诗模型。但是阿里云的GPU服务器特别贵，一块8G显存的Nvidia P4显卡就要8万一年左右，成本太高，可以买配置不错的多卡物理机了，不过土豪可以忽略。所以这里采用本地GPU物理机来跑写诗模型，在Nvidia RTX2080Ti不考虑并发的条件下，平均单次写诗时间在300-600ms左右，但是本地物理机没有公网IP，云服务器无法直接访问，这里采用了[autossh反向代理穿透到内网](https://www.jianshu.com/p/7accc1e485d3
)，把本地GPU服务器的端口映射到云服务器上，这样云服务器就可以访问本地物理机端口服务了。

```
autossh -M 5686 -fCNR *:8888:localhost:11456 user@xxx.xxx.xxx.xxx
```
> 其中，xxx.xxx.xxx.xxx 是云服务器的ip，user是用户账号， 8888是云服务器访问端口，可以自行设定，11456是本地GPU服务器flask监听的端口，这样就可以在云服务器上通过访问 127.0.0.1:8888既可以访问本地GPU服务器flask发布的11456服务

**2、模型**   

从模型角度优化时间，可以从两块进行考虑，一块选择较小的模型，另一块对BeamSearch进行优化，下面详细介绍：  

目前选择的预训练模型是RoBERTa, 如果需要缩短时间，可以选择albert, RoBERTa-small, RoBERTa-tiny, 我用RoBERTa-small尝试了一下，预测时间和权重文件缩小了一倍，并发可以提升四倍，但是我让我诗词界的朋友看了一下写诗质量，下降太多，我就放弃了，还是选择RoBERTa来训练写诗模型；    

另一块就是seq2seq的BeamSearch优化，模型生成诗是一个字一个字预测的，每次用Topk个候选集预测，再选择当前最优的Topk个候选集作为下一次预测的输入，直到碰到截止符为止，所以TopK越大，相应的预测时间就越长，生成诗的质量也相对会好一点，但是不绝对。总得来说通过BeamSearch生成的诗质量相对好一点，但是比较单一，多样性比较差。所以我想了一个办法，每次从topk里面按loss概率分布随机选一个值，作为下一次预测的输入，这样作诗时间就不会受topk的影响，同时增加了生成古诗的多样性。我个人感觉挺好，我把生成的诗给我诗词界的朋友又看了一下，他说作诗质量不如之前的好，我问他你喜欢之前的还是现在多样性丰富的作诗机，他说他还是更看重作诗的质量。这时，我苦恼了，不知道怎么权衡好，因为之前beamsearch的topk大于8时，时间会到达好几秒，不仅影响用户体验，同时增加服务器压力，减少并发能力。有一天晚上，我突然灵机一动，想到了一个折中的方法，就是前端输入的topk小于等于3时，采用传统的beamsearch生成古诗；当大于3时，采用第二种随机选择的方法。这样既能保证作诗质量，同时能增加生成古诗的多样性，而且还不会太多增加单次作诗的平均时间，保证作诗网站的并发能力。  

```
if topk <= 3:
  result = self.poem_gen_sent(text, topk=topk)
else:
  result = self.poem_gen_sent_random(text, topk=topk)
```

**3、后端**  

由于python flask多线程效率很低，采用gunicron多进程提升并发性能，可以创建一个shell脚本文件server_poem.sh
```
gunicorn -w $1 -b 0.0.0.0:11456 server_poem:app
```
执行
```
sh server_poem.sh 9
```
就可以启动服务了，9是进程数量，由于每个进程都是独立占用显存的，具体进程数多少需要看单个进程占用的显存及显卡总显存大小，总之所有进程占用的显存加起来不能超过总显存。   

如果本地有多台GPU服务器，可以通过[程序文件共享](https://www.cnblogs.com/woxbwo/p/11581826.html)，再用上面的方法各自开服务，然后通过autossh 映射到云服务器不同的端口，云服务器再通过nginx做负载均衡，比如本来有三台GPU服务器，  

| 服务器 | 本地监听端口  | 反向代理穿透脚本 | 云服务器访问端口 |
| - | - | - | - |
| GPU服务器1 | 0.0.0.0:11456 | autossh -M 5686 -fCNR \*:8888:localhost:11456 user@xxx.xxx.xxx.xxx | 127.0.0.1:8888 |      
| GPU服务器2 | 0.0.0.0:11456  | autossh -M 5686 -fCNR \*:8889:localhost:11456 user@xxx.xxx.xxx.xxx | 127.0.0.1:8889 |
| GPU服务器3 | 0.0.0.0:11456  | autossh -M 5686 -fCNR \*:8890:localhost:11456 user@xxx.xxx.xxx.xxx | 127.0.0.1:8890 |

这样三台本地GPU服务器的服务，就通过反向代理穿透映射到云服务器8888,8889,8890三个端口上，但是对用户来说，到底应该访问哪个端口呢？别急，下面可以用nginx做负载均衡

```
upstream poemapi.com {
    server 127.0.0.1:8888 weight=1;
    server 127.0.0.1:8889 weight=4;
    server 127.0.0.1:8890 weight=2;
}

server {
    listen 12575;
    server_name _;
    location / {
        proxy_pass http://poemapi.com/zqcloudapi/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
如果本地三台GPU服务器的显卡性能，开得进程数量不一样，可能根据实际情况设置不同的权重，性能好的weight可以大一点。完成上述配置，重启nginx服务，就可以在云服务器访问 127.0.0.1:12575 对三台本地GPU服务器的服务做负载均衡了。


**4、接口**  

由于某些热门主题会被不同的用户重复请求，如果每次都重复计算的话，会占用大量的GPU服务器资源，所以这里引入了redis来保存已生成的诗，为了让做的诗有多样性，这里没有永久保存，只保存了一天，保存KEY为coupletpoem_input_topk, 其中input和topk为前端输入。

再由于python的效率很低，就是不做任务处理，python flask的并发请求效率也不高，所以最后只让python flask专业做写诗的事情，其他鸡毛蒜皮琐事的事情就不用python管了，这里引入异步非阻塞的node来面向前端做接口，当然也可以用java，本人只是觉得java比较重，就选择了轻量级的node，没有复杂业务逻辑的并发性能也不亚于java。node接口判断的业务逻辑也很简单，首先判断redis里面有没有值，有值就直接返回给前端，没有值调用上面负载均衡映射的python flask API生成诗词，返回给前端并保存到redis，同时把请求日志放到队列，由另外一个独立的线程定时专门把业务请求日志存储到mongodb数据库。

**5、前端**  

前端同一个用户同一个主题同一个topk可能会请求多次，为了减少后端的压力，前端可以把请求到的结果存储到localstorge里面，存储周期根据实际场景设定，这里设了一个小时，这样第二次请求的时候，直接从localstorage取数据既可以，这个功能可以封装在接口程序里，参考如下：

```
export const getServerData = (url, method, object, period, callback, errcallback) => {
  url = Config.host + url;
  const key = md5(url + method + JSON.stringify(object));
  // 判断本地localstorage数据是否有效
  const data = getDataFromLocalStorage(key, period);
  if (!data) {
    getData(url, method, object, text => {
      const resultobj = JSON.parse(text);
      // console.log(resultobj);
      if (resultobj.errcode === 0) {
        if (period > 0) {
          resultobj.result.time = new Date().getTime(); // 添加当前时间
          localStorageUtil.save(key, resultobj.result);
        }
        callback(resultobj.result);
      } else {
        // console.log('业务逻辑异常:', resultobj.errmsg);
        errcallback('业务逻辑异常: ' + resultobj.errmsg);
      }
    }, err => {
      errcallback(err);
    });
  }
  else {
    callback(data);
  }
};
```
如果这个接口明文传输的话，可能会被第三方的爬虫脚本盯上，大大增加服务器的压力，影响前端的写诗性能。所以需要对接口的参数和返回结果进行加密，比如参数可以采用AES加密，返回结果可以采用base64 + AES + DES多级加密，每个的key和iv都可以不一样，提升破解难点。

### 敏感词过滤
在大陆上线带文本输入的AI产品，这一步非常重要，秉着宁可错杀1000，也尽量不要遗漏一个的原则，要不然容易被请去喝茶，请自重！   

如果有钱的话可以直接调用[百度文本审核API](https://ai.baidu.com/tech/textcensoring)，这个大概1.5分一条，个人实名认证可以赠送5万条，企业实名认证可以赠送50万条，由于我多的时候每天有10w+次请求，这点赠送量是不够的，下面是我做敏感词过滤的几道防线，供参考：   

1、国家现核心领导人前领导人名字及其他核心敏感词拼音级过滤，如果用汉字，如细净瓶，吸金瓶，供惨谠等等，字太多很难遍历全，还是拼音过滤省事一点； 

> npm 有拼音库，安装后可以直接调用，把汉字转换成拼音，详见[NPM汉字拼音转换工具](https://www.npmjs.com/package/pinyin)介绍。
```
npm install pinyin
```

2、国家现核心领导人前领导人名字及其他核心敏感词包含过滤，只要输入的文本里面出现这几个字，无论什么顺序，中间夹杂了多少个字全部过滤，如习惯平易近人，习惯禁止评论，长江恩泽于民众等等，虽然这样可能会错杀一些输入文本，但是总比遗漏一个请去喝茶要好一点；   
3、某些相近的字也需要过滤，如习和刁等；   
4、国家核心领导人去过的某些地方，吃过的某些东西，说过的某些话，做过的某些事情及长的像的某些东西等等都要过滤，比如庆丰包子，蛤膜，维尼熊等等；      
5、根据第三方开源的[敏感词库](https://github.com/fighting41love/funNLP/tree/master/data/%E6%95%8F%E6%84%9F%E8%AF%8D%E5%BA%93)进行过滤；  
6、定期调用百度文本审核API标注日志中的样本，并用albert训练自己的敏感词二分类识别模型，不需要GPU，可以在云服务器的CPU上跑，每条文本大概6-7ms左右，判别过的就可以放到Redis缓存里，不需要重复判断。具体模型训练方式可以参考苏老师bert4keras的[情感二分类例子](https://github.com/bojone/bert4keras/blob/master/examples/task_sentiment_albert.py)。

上述前五条都可以在前端完成，如果校验不通过直接提示“您输入的主题无法作诗，请重新输入”，不需要增加服务端的压力，前端校验通过后，第六条需要在后端完成，再加一道保险。前端过滤代码参考如下：

```
export const checkKeys = (text) => {
  let flag = true;
  // 根据白名单判断
  if (keyobj['whitekeys'].indexOf(text) >= 0) {
    return true;
  }
  // 剔除非法字符
  text = text.replace(/[\s+\-\.\。\，\；\?\？\！\!\…\+\!\@\#\$\%\^\&\*()\（\）\￥\[\]]/g,"");
  if (words.indexOf(text) >=0) {
    return false;
  }

  // 转换成拼音并判断，如果文本中包含核心敏感词的拼音，就返回false
  let py = pinyin(text, {
    style: pinyin.STYLE_NORMAL
  });
  py = py.join('');

  for (let i = 0; i<pinyins.length; i++) {
    if (py.indexOf(pinyins[i])>-1) {
      console.log(text, py, pinyins[i]);
      return false;
    }
  }

  py = pinyin(text);
  py = py.join('');
  console.log(py);

  for (let i = 0; i<pinyins2.length; i++) {
    if (py.indexOf(pinyins2[i])>-1) {
      console.log(text, py, pinyins2[i]);
      return false;
    }
  }

  // 根据敏感词判断，只要输入文本包含敏感词就返回false
  for (let i = 0; i < keyobj['keys'].length; i++) {
    if (text.indexOf(keyobj['keys'][i])>=0) {
      console.log(text, keyobj['keys'][i]);
      flag = false;
      return false;
    }
  }

  // 根据核心敏感词判断，只要输入出现敏感词这几个字，不论什么顺序，中间夹了多少个字，都返回false
  for (let i = 0; i < keyobj['subkeys'].length; i++) {
    let j = 0;
    let item = keyobj['subkeys'][i];
    let newtext = text;
    for (j = 0; j < item.length; j ++) {
      if (newtext.indexOf(item[j]) < 0){
        break;
      }
      newtext = newtext.replace(item[j],'');
    }
    if (j == item.length) {
      return false;
    }
  }
  return flag;
};
```


### 运营推广SEO优化
上线之后就可以推广了，首先是在各大诗词群里面推广，然后在贴吧，论坛，知乎，CSDN，豆瓣等平台发帖，大家觉得好玩之后，也会自己在常用的平台上发帖，让更多的人参与进来。  
接下来就是要通过SEO优化了，提升百度、谷歌、搜狗、360、神马等搜索引擎的搜索排名，通过搜索引擎来导流，主要包括以下几点：
- 优化首页meta标签的标题，关键词及描述，尽量能包含你想要的搜索关键词，比如我想要的关键词就是AI写诗，在线作诗机，藏头诗生成器，自动对联等，就可以着重描述这些关键词；
- 和高权重的网站相互友情链接，需要发邮件和网站负责人一个个联系，互挂友情链接，网站权重可以在[这里](https://www.aizhan.com/)查询；
- 在贴吧，论坛，知乎，CSDN，豆瓣等平台多发帖，并附上网站链接，增加外链数量   

以上的做法都是为了提升网站权重，从而提升搜索引擎的搜索权重，提升排名，增加流量，如果不差钱的话当然也可以选择付费推广，用户更精准成效也更快。

### 生成效果
![avatar](诗三百效果图.png)

**七律·咏梅**  
一枝清瘦倚疏篱，不与群花较等差。  
雪里有香浑是玉，月中无影更多葩。  
孤山处士曾题品，姑射仙人合赋家。  
莫道广平心似铁，也能吟咏到昏鸦。  

**藏头诗·不负韶华**   
不见当年李谪仙，  
负图千古恨绵延。  
韶华过眼成陈迹，  
华表空留鹤一传。  

**蝶恋花·柳絮**  
一片闲愁随水去。桃李无言，不觉春光暮。官柳倡狂遮大路。游丝莫系相思树。  
人比花枝能解语。镜里蛾眉，犹是当初否。心似行云年似雾。东风又绿罗巾露。

**上联**：虎啸青山抒壮志  
**下联**：龙腾碧海展宏图

**上联**：望江楼上望江流，江楼千古，江流千古  
**下联**：观海岛中观海涌，海岛万家，海涌万家


上面是诗三百·人工智能诗歌写作平台，从模型训练到前后端开发到优化推广的所有过程，如果大家觉得写得好，欢迎给个star，有什么不足的地方也欢迎给出宝贵的建议！

### 相关网站：
- 诗三百·人工智能诗歌写作平台：   https://www.aichpoem.net/  
- 清华九歌·人工智能诗歌写作系统： http://jiuge.thunlp.org/jueju.html  
- 华为乐府·人工智能作诗小程序：   微信公众号搜索 “乐府AI”  
- 微软小冰·AI现代诗歌创作系统：   https://poem.msxiaobing.com/  
- 微软亚洲研究院·电脑对联系统：   https://duilian.msra.cn/app/couplet.aspx
- 王斌·人工智能自动对对联系统：   https://ai.binwang.me/couplet/

### 特别鸣谢：
- [科学空间](https://spaces.ac.cn/)博主苏剑林老师开源[bert4keras](https://github.com/bojone/bert4keras/)
- Werneror老师开源85万首[古诗词数据集](https://github.com/Werneror/Poetry)
- 王斌老师开源70万首[对联数据集](https://github.com/wb14123/couplet-dataset)

### 友情赞助
如果您觉得诗三百对您作诗和学习有帮助，欢迎通过打赏方式支持开发者，您的支持是开发者不断完善优化的巨大动力🙏🏻

<img style="width:300px;max-width:100%" src="http://www.aichpoem.net/static/img/wx_mcode.jpeg" />


### 合作建议：
- QQ: 540629297  (备注诗三百) 
- 微信: wangjiezju (备注诗三百) 
- E-mail: wangjiezju@163.com

