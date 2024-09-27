API文档

感谢您使用JoinQuant（聚宽）量化平台，以下内容主要介绍聚宽量化平台的API使用方法，目录中带有"♠" 标识的API是 "回测环境/模拟"的专用API，不支持在投资研究模块中调用。

内容较多，可使用Ctrl+F进行搜索。

如果以下内容仍没有解决您的问题，请您查看常见问题、常见Bug或者警告解决方法或者通过社区提问的方式告诉我们。此外您也可以通过在线客服免费咨询
img

开始写策略

简单但是完整的策略

先来看一个简单但是完整的策略:

def initialize(context):
    # 定义一个全局变量, 保存要操作的股票
    g.security = '000001.XSHE'
    # 运行函数
    run_daily(market_open, time='every_bar')

def market_open(context):
    if g.security not in context.portfolio.positions:
        order(g.security, 1000)
    else:
        order(g.security, -800)
一个完整策略只需要两步:

设置初始化函数: initialize,上面的例子中, 只操作一支股票: '000001.XSHE', 平安银行
实现一个函数, 来根据历史数据调整仓位.
这个策略里, 每当我们没有股票时就买入1000股, 每当我们有股票时又卖出800股, 具体的下单API请看order函数.

这个策略里, 我们有了交易, 但是只是无意义的交易, 没有依据当前的数据做出合理的分析

下面我们来看一个真正实用的策略

实用的策略

在这个策略里, 我们会根据历史价格做出判断:

如果上一时间点价格高出五天平均价1%, 则全仓买入
如果上一时间点价格低于五天平均价, 则空仓卖出
# 导入聚宽函数库
import jqdata

# 初始化函数，设定要操作的股票、基准等等
def initialize(context):
    # 定义一个全局变量, 保存要操作的股票
    # 000001(股票:平安银行)
    g.security = '000001.XSHE'
    # 设定沪深300作为基准
    set_benchmark('000300.XSHG')
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)
    # 运行函数
    run_daily(market_open, time='every_bar')

# 每个单位时间(如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次)调用一次
def market_open(context):
    security = g.security
    # 获取股票的收盘价
    close_data = attribute_history(security, 5, '1d', ['close'])
    # 取得过去五天的平均价格
    MA5 = close_data['close'].mean()
    # 取得上一时间点价格
    current_price = close_data['close'][-1]
    # 取得当前的现金
    cash = context.portfolio.available_cash

    # 如果上一时间点价格高出五天平均价1%, 则全仓买入
    if current_price > 1.01*MA5:
        # 用所有 cash 买入股票
        order_value(security, cash)
        # 记录这次买入
        log.info("Buying %s" % (security))
    # 如果上一时间点价格低于五天平均价, 则空仓卖出
    elif current_price < MA5 and context.portfolio.positions[security].closeable_amount > 0:
        # 卖出所有股票,使这只股票的最终持有量为0
        order_target(security, 0)
        # 记录这次卖出
        log.info("Selling %s" % (security))
    # 画出上一时间点价格
    record(stock_price=current_price)
策略引擎介绍

安全

保证您的策略安全是我们的第一要务
在您使用我们网站的过程中, 我们全程使用https传输
策略会加密存储在数据库
请不要在其他非聚宽平台登录聚宽账号。
回测时您的策略会在一个安全的进程中执行, 我们使用了进程隔离的方案来确保系统不会被任何用户的代码攻击, 每个用户的代码都运行在一个有很强限制的进程中:
只能读指定的一些python库文件

不能写和执行任何文件, 如果您需要保存和读取私有文件, 请看write_file/read_file

限制了cpu和内存, 堆栈的使用，当使用多核时，回测耗时按CPU占用耗时计算

可以访问网络, 但是对带宽做了限制, 下载最大带宽为500Kbps, 上传带宽为10Kbps

有严格的超时机制, 如果handle_data超过30分钟则立即停止运行 对于读取回测所需要的数据, 和输出回测结果, 我们使用一个辅助进程来帮它完成, 两者之间通过管道连接.

我们使用了linux内核级别的apparmer技术来实现这一点. 有了这些限制我们确保了任何用户不能侵入我们的系统, 更别提盗取他人的策略了.

数据
股票数据：我们拥有所有A股上市公司2005年以来的股票行情数据、财务数据、上市公司基本信息、融资融券信息等。为了避免幸存者偏差，我们包括了已经退市的股票数据。其中volume（成交量）字段单位是股。
基金数据：我们目前提供了600多种在交易所上市的基金的行情、净值等数据，包含ETF、LOF、分级A/B基金以及货币基金的完整的行情、净值数据等，请点击基金数据查看。
金融期货数据：我们提供中金所推出的所有金融期货产品的行情数据，并包含历史产品的数据。
股票指数：我们支持近600种指数数据，包括指数的行情数据以及成分股数据。为了避免未来函数，我们支持获取历史任意时刻的指数成分股信息，具体见get_index_stocks。
行业板块：我们支持按行业、按板块选股，具体见get_industry_stocks
概念板块：我们支持按概念板块选股，具体见get_concept_stocks
所有的行情数据我们均已处理好前复权信息。
我们当日的回测数据会在收盘后通过多数据源进行校验，并在T+1（第二天）的00:01更新。
我们提供的所有行情K线数据为后对齐，标识K线的时间为数据的结束时间。在一分钟K线上，没有09:30，从09:31开始，有15:00的K线，共计240根。表示时间为09:31的一分钟K线，其数据时间为09:25:00~09:30:59，这一分钟的开盘价是09:25的集合竞价的价格。
期货K线的划分方式：将标的当天的开盘时间到收盘时间的日历时间按照划分单位划分区间，然后将同一个区间的分钟bar合并。例如，某个标的的开盘时间为09:30,收盘时间为15:00，然后划分单位为5m，划分bar的逻辑如下：将09:30-15:00按照5m划分区间，然后将这个标当天行情在同一个区间的分钟bar合并。
运行频率

聚宽支持天、分钟及tick频率，其他频率您可以在此基础上自己根据时间判断，有关运行频率解析及隔固定时间运行方法请点击查看，下面是运行频率的详细说明。

1. Bar 的概念

在一定时间段内的时间序列就构成了一根 K 线（日本蜡烛图），单根 K 线被称为 Bar。

如果是一分钟内的 Tick 序列，即构成一根分钟 K 线，又称分钟 Bar;
如果是一天内的分钟序列，即构成一根日线 K 线，又称日线 Bar;

Bar 的示意图如下所示：

Bar 的示意图

Bar 就是时间维度上，价格在空间维度上变化构成的数据单元。如下图所示，多个数据单元 Bar 构成的一个时间序列。

K线序列

2. 频率详解

下列图片中齿轮为 handle_data(context, data) 的运行时间，before_trading_start(context) 等其他函数运行时间详见相关API。

频率：天

当选择天频率时， 算法在每根日线 Bar 都会运行一次，即每天运行一次。

在算法中，可以获取任何粒度的数据。

日K线

频率：分钟

当选择分钟频率时， 算法在每根分钟 Bar 都会运行一次，即每分钟运行一次。

在算法中，可以获取任何粒度的数据。

分钟K线

频率：Tick

当选择 Tick 频率时，每当新来一个 Tick，算法都会被执行一次。

执行示意图如下图所示：

Tick序列


运行时间
设置您的策略什么时候运行，主要由设置策略频率（天、分钟或者tick）与控制策略运行时间的API共同完成

开盘前(9:00)运行:

run_monthly/run_weekly/run_daily中指定time='09:00'运行的函数
before_trading_start
盘中运行:

run_monthly/run_weekly/run_daily中在指定交易时间执行的函数, 执行时间为这分钟的第一秒. 例如: run_daily(func, '14:50') 会在每天的14:50:00(精确到秒)执行
handle_data
按日回测/模拟, 在9:30:00(精确到秒)运行, data为昨天的天数据
按分钟回测/模拟, 在每分钟的第一秒运行, 每天执行240次, 不包括11:30和15:00这两分钟, data是上一分钟的分钟数据. 例如: 当天第一次执行是在9:30:00, data是昨天14:59至15:00这一分钟的分钟数据, 当天最后一次执行是在14:59:00, data是14:58至14:59:00这一分钟的分钟数据.
收盘后(15:00后半小时内)运行:

run_monthly/run_weekly/run_daily中指定time='15:30'运行的函数
after_trading_end
同一个时间点, 总是先运行 run_xxx 指定的函数, 然后是 before_trading_start , handle_data 和 after_trading_end

注意:

为了避免您换算错误，建议设置time为具体的时间（例如：time='9:30'）;
run_xxx指run_monthly/run_weekly/run_daily中的任意一个
run_xxx 指定的函数只能有一个参数 context, data 不再提供, 请使用 history等获取;
initialize / before_trading_start / after_trading_end / handle_data 都是可选的, 如果不是必须的, 不要实现这些函数, 一个空函数会降低运行速度;
run_xxx和handle_data不要在同一个策略中使用，建议使用run_xxxx;
一个策略中可以写多个run_xxx函数，例如需要每分钟运行和定时运行的话，可以这样设置：
## func1, func2, func3都是您自己实现的函数
# 每分钟运行     
run_daily(func1, time='every_bar')      
# 11:00定时运行        
run_daily(func2, time='11:00')     
# 14:00定时运行        
run_daily(func3, time='14:00')  
# 14:50定时运行        
run_daily(func2, time='14:50')      
订单处理
订单处理总体过程：

从委托到成交的流程:
订单创建->订单检查->报单->确认委托->撮合，在订单检查时未通过则订单取消；
回测模式整个过程是同步进行，order_*函数执行成功后会创建一个order对象,市价单的order对象撮合后(交易时间下单立即撮合)会立即获得交易的结果,限价单的order对象每次成交后都会立即更新；
目前官网的模拟盘过程同回测；
所有未完成订单将在本交易日结束后撤销。
运行频率	委托类型	关闭盘口撮合	启用盘口撮合
天	市价单	按最新价+滑点撮合	按盘口撮合
限价单	下单时尝试按最新价+滑点撮合，剩余部分挂单每分钟尝试按分钟Bar撮合	下单时按盘口撮合，剩余部分挂单每分钟尝试按分钟Bar撮合
分钟	市价单	按最新价+滑点撮合	按盘口撮合
限价单	下单时尝试按最新价+滑点撮合，剩余部分挂单每分钟尝试按分钟Bar撮合	下单时按盘口撮合，剩余部分挂单每分钟尝试按分钟Bar撮合
tick	市价单	按最新价+滑点撮合	按盘口撮合
限价单	下单时尝试按最新价+滑点撮合，剩余部分挂单每tick按tick撮合	下单时按盘口撮合，剩余部分挂单每tick按tick撮合
撮合流程
在回测和模拟交易中，所有的委托（无论市价单还是限价单）在下单后都将尝试进行撮合，撮合的逻辑根据是否打开盘口撮合进行选择；
若委托未能完全成交且为限价单，则挂单，然后根据挂单撮合逻辑进行后续的流程。
下单时的撮合逻辑
未启用盘口撮合时
当 “最新价+滑点” 在涨跌停范围内将尝试进行撮合，若满足以下条件则成交，成交价为最新价+滑点：

买入/开多/平空时，委托价 >= 最新价+滑点；
卖出/开空/平多时，委托价 <= 最新价-滑点；
若标的“最新价+滑点”不低于涨停或者不高于跌停时：

跌停时市价卖单会被撤销，涨停时市价买单会被撤销；
限价单会挂单等待撮合。
交易价格: 最新价 + 滑点，如果在开盘时刻运行， 最新价格为开盘价。 其他情况下， 为上一分钟的最后一个价格或上一个tick的最新价。

满足撮合条件时成交量的限制：

回测：不超过当日总成交量 * order_volume_ratio；
模拟交易：全部成交；
超出最大成交量的部分：

对于市价单，剩余部分将撤单；
对于限价单，将按委托价挂单，根据策略频率使用不同的后续逻辑。
注意:
回测中可通过选项 order_volume_ratio 设置每日最大的成交量, 例如: 0.25 表示下单成交量不会超过本日成交量的 25%；
通过选项 order_volume_ratio 设置每日最大的成交量仅限制了每个订单的成交量, 虽然你可以通过多次下单来超过该限制, 但是为了对你的回测负责请不要这么做；
context.portfolio 中的持仓价格会使用上一分钟的最后一个价格更新。
启用盘口撮合时
仅在模拟交易中可启用盘口撮合。

根据对手盘盘口进行撮合；
优先从一档（买一/卖一）开始撮合，根据成交量算出加权均价；
盘口撮合不限制成交量；
若无盘口数据，使用未启用盘口撮合时的逻辑尝试进行撮合；
当盘口无法完全成交时：
对于市价单，剩余部分将全部按最高一档盘口撮合；
对于限价单，将按委托价挂单，根据策略频率使用不同的后续逻辑；
对涨跌停情况的处理：
若涨跌停时存在对手盘，则按盘口正常撮合；
若没有对手盘，此时转入未启用盘口时的撮合逻辑：跌停时市价卖单会被撤销，涨停时市价买单会被撤销；限价单会挂单等待撮合。
挂单时的撮合逻辑
针对不同频率（天/分钟，tick）的策略，挂单的限价单将有以下撮合逻辑

按分钟Bar撮合
天/分钟频率的策略，挂单的限价单每分钟都将尝试在本分钟Bar结束时按照Bar信息尝试进行撮合，若满足以下条件则成交，成交价为委托价：
买入/开多/平空时，委托价 > Bar的最低价；
卖出/开空/平多时，委托价 < Bar的最高价；
成交量限制：
模拟交易中不超过本分钟Bar成交量；
回测中不超过本分钟Bar成交量 * order_volume_ratio；
若订单未完全成交，剩余部分将在每个分钟bar结束时继续尝试撮合直到全部成交或收盘；
按tick撮合
挂单的限价单每个tick都将尝试在tick结束时按照tick信息尝试进行撮合，若满足以下条件则成交，成交价为委托价：
买入/开多/平空时，若委托价 > tick的最新价，则进行撮合；
卖出/开空/平多时，若委托价 < tick的最新价，则进行撮合；
成交量限制：按tick撮合时不检查成交量，当出现满足撮合条件的价格后，剩余部分全部以委托价成交。
非交易时段下单的特别说明
如果非交易时间下单且不撮合，不管是市价单还是限价单，都会挂单
对于日频级策略，会在开盘时尝试进行撮合；对于分钟或者tick频率的策略，会在下一个分钟bar或者tick完成时尝试撮合
市价单挂单后开始交易时会按照下单时的逻辑撮合，限价单按照bar/tick来撮合
如果用户在 11:30:01 下单，那么订单挂单，引擎退出；下午开盘前引擎恢复运行时，账户初始化会定位未完成订单的对应频率的数据，如果是tick，可能会影响用户的tick订阅数量
图解订单处理

拆分合并与分红
传统前复权回测模式：当股票发生拆分，合并或者分红时，股票价格会受到影响，为了保证价格的连续性, 我们使用前复权来处理之前的股票价格，给您的所有股票价格已经是前复权的价格。
真实价格（动态复权）回测模式：当股票发生拆分，合并或者分红时，会按照历史情况，对账户进行处理，会在账户账户中增加现金或持股数量发生变化，并会有日志提示。
内容
传统前复权回测模式 与 真实价格（动态复权）回测模式 区别
是否开启动态复权(真实价格)模式对模拟交易的影响
股息红利税的计算
真实的税率计算方式如下：

分红派息的时候，不扣税；
等你卖出该只股票时，会根据你的股票持有时间（自你买进之日，算到你卖出之日的前一天，下同）超过一年的免税。2015年9月之前的政策是，满一年的收5%。现在执行的是,2015年9月份的新优惠政策：满一年的免税；
等你卖出股票时，你的持有时间在1个月以内（含1个月）的，补交红利的20%税款，券商会在你卖出股票当日清算时直接扣收；
等你卖出股票时，你的持有时间在1个月至1年间（含1年）的，补交红利的10%税款，券商直接扣；
分次买入的股票，一律按照“先进先出”原则，对应计算持股时间；
当日有买进卖出的（即所谓做盘中T+0），收盘后系统计算你当日净额，净额为买入，则记录为今日新买入。净额为卖出，则按照先进先出原则，算成你卖出了你最早买入的对应数量持股，并考虑是否扣税和税率问题。
在回测及模拟交易中，由于需要在分红当天将扣税后的分红现金发放到账户，因此无法准确计算用户的持仓时间（不知道股票卖出时间），我们的计算方式是，统一按照 20% 的税率计算的。


滑点
在实战交易中，往往最终成交价和预期价格有一定偏差，因此我们加入了滑点模式来帮助您更好地模拟真实市场的表现，可以使用set_slippage设置滑点。


交易税费
交易税费包含券商手续费和印花税。您可以通过set_order_cost来设置具体的交易税费的参数。

券商手续费
中国A股市场目前为双边收费，券商手续费系默认值为万分之三，即0.03%，最少5元。

印花税
印花税对卖方单边征收，对买方不再征收，系统默认为千分之一，即0.1%。


风险指标
风险指标数据有利于您对策略进行一个客观的评价。

注意: 无论是回测还是模拟, 所有风险指标(年化收益/alpha/beta/sharpe/max_drawdown等指标)都只会每天于17:00左右更新一次, 也只根据每天收盘后的收益计算, 并不考虑每天盘中的收益情况. 例外:

分钟和TICK模拟盘每分钟会更新策略收益和基准收益
按天模拟盘每天开盘后和收盘后会更新策略收益和基准收益
基准收益的计算起点取的是回测开始时间前一个交易日的收盘价
那么可能会造成这种现象: 模拟时收益曲线中有回撤, 但是 max_drawdown 可能为0.

名称 描述
Total Returns 策略收益
Total Annualized Returns 策略年化收益
Alpha 阿尔法
Beta 贝塔
Sharpe 夏普比率
Sortino 索提诺比率
Information Ratio 信息比率
Algorithm Volatility 策略波动率
Benchmark Volatility 基准波动率
Max Drawdown 最大回撤
Downside Risk 下行波动率
胜率 胜率(%)
日胜率 日胜率(%)
盈亏比 盈亏比
AEI 日均超额收益
超额收益最大回撤 超额收益最大回撤
超额收益夏普比率 超额收益夏普比率
超额收益 除法版超额收益率说明
对数轴 对数轴说明
对数轴上的超额收益 对数轴上的超额收益的计算方法

回测环境
回测引擎可在Python2.7与Python3.6上运行,默认使用Python3.6。我们将在未来逐步停止对Python2.7引擎的更新，强烈建议您使用Python3.6开发策略。
我们支持所有的Python标准库和部分常用第三方库, 具体请看: 研究和回测（模拟）中都支持哪些第三方Python库. 另外您可以把.py文件放在研究根目录, 回测中可以直接import, 具体请看: 自定义python库
安全是平台的重中之重, 您的策略的运行也会受到一些限制, 具体请看: 安全

回测过程
准备好您的策略, 选择要操作的股票池, 实现handle_data函数
选定一个回测开始和结束日期, 选择初始资金、调仓间隔(每天还是每分钟), 开始回测
引擎根据您选择的股票池和日期, 取得股票数据, 然后每一天或者每一分钟调用一次您的handle_data函数, 同时告诉您现金、持仓情况和股票在上一天或者分钟的数据. 在此函数中, 您还可以调用函数获取任何多天的历史数据, 然后做出调仓决定.
当您下单后, 我们会根据接下来时间的实际交易情况, 处理您的订单. 具体细节参见订单处理
下单后您可以调用get_open_orders取得所有未完成的订单, 调用cancel_order取消订单
您可以在handle_data里面调用record()函数记录某些数据, 我们会以图表的方式显示在回测结果页面
您可以在任何时候调用log.info/debug/warn/error函数来打印一些日志
回测结束后我们会画出您的收益和基准(参见set_benchmark)收益的曲线, 列出每日持仓,每日交易和一系列风险数据。

模拟盘注意事项
模拟交易是根据回测的策略创建的，因此需要先回测，再创建模拟交易，如何创建模拟交易？

为了避免以不合理的价格对标的进行下单，模拟盘在下单时会检查开盘（例如股票为9:25）到下单时刻的累积成交量，若为0则会拒绝，提示：WARNING - 该标的截至到目前成交量为0 ，暂时无法成交。

模拟盘中因尽量避免在距离开盘时间较早的时间点进行下单, 比如9点以前对股票下单，可能导致当时还没有拿到涨跌停价而产生比较异常的委托甚至委托失败。

在日级模拟中使用时，使用handle_data或者run_daily中time='9:30'，策略的实际运行时间是9:27~9:30之间；股指期货在9:27~9:30之间有可能没有产生集合竞价，会出现9:30下单提示该标的截至到目前成交量为0，可以忽略或者在9:31及之后运行；

模拟盘在每天运行结束后会保存状态, 结束进程(相当于休眠). 然后在第二天恢复.

进程结束时会保存这些状态:

用户账户, 持仓

使用 pickle 保存 g 对象. 注意

g 中以 '__' 开头的变量将被忽略, 不会被保存
g 中不能序列化的变量不会被保存, 重启后会不存在. 如果你写了如下的代码:
def initialize(context):
    g.query = query(valuation)
g 将不能被保存, 因为 query() 返回的对象并不能被持久化. 重启后也不会再执行 initialize, 使用 g.query 将会抛出 AttributeError 异常。正确的做法是, 在 process_initialize 中初始化它, 并且名字以 '__' 开头.
def process_initialize(context):
    g.__query = query(valuation)
注意: 涉及到IO(打开的文件, 网络连接, 数据库连接)的对象是不能被序列化的:
query(valuation) : 数据库连接
open("some/path") : 打开的文件
requests.get('') : 网络连接
使用 pickle保存 context 对象, 处理方式跟 g 一样

为了防止恶意攻击, 序列化之后的状态大小不能超过 30M, 如果超出将在保存状态时运行失败. 当超过 20M 时日志中会有警告提示, 请注意日志.

恢复过程是这样的:

加载策略代码, 因为python是动态语言, 编译即运行, 所以全局的(在函数外写的)代码会被执行一遍.
使用保存的状态恢复 g, context, 和函数外定义的全局变量.
如果策略代码和上一次运行时发生了修改，而且代码中定义了 after_code_changed 函数，则会运行 after_code_changed(#after_code_changed) 函数。
执行 process_initialize, 每次启动时都会执行这个函数.
重启后不再执行 initialize 函数, initialize 函数在整个模拟盘的生命周期中只执行一次. 即使是更改回测后, initialize 也不会执行.

模拟盘更改回测之后上述的全局变量(包括 g 和 context 中保存的)不会丢失. 新代码中 initialize 不会执行. 如果需要修改原来的值, 可以在 after_code_changed 函数里面修改, 比如, 原来代码是:

def initialize(context):
    g.stock = '000001.XSHE'
代码改成:

def initialize(context):
    g.stock = '000002.XSHE'
执行时, g.stock 仍然是 '000001.XSHE', 要修改他们的值, 必须定义 after_code_changed:

def after_code_changed(context):
    g.stock = '000002.XSHE'
创建模拟交易时, 如果选择的日期是今天, 则从今天当前时间点开始运行, 应该在当前时间点之前运行的函数不再运行. 比如: 今天10:00创建了按天的模拟交易, 选择日期是今天, 代码中实现了 handle_data 和 after_trading_end, 则 handle_data 今天不运行, 而 after_trading_end 会在 15:30 运行

当模拟交易在A时间点失败后, 然后在B时间点"重跑", 那么 A-B 之间的时间点应该运行的函数不再运行

因为模拟盘资源有限, 为了防止用户创建之后长期搁置浪费资源, 我们做出如下限制: 如果一个模拟盘同时满足下面条件, 则暂缓运行:

模拟盘所有者在最近一个月内没有访问过聚宽网站
当用户重新使用网站后, 第二天会继续运行(会把之前的交易日执行一遍, 并不会跳过日期)

由于模拟盘资源有限及防止恶意攻击, 我们设置：（1）模拟交易序列化之后的状态大小不能超过 30M；（2）每个函数运行时间不能超过1800s；（3）进程占用内存不能超过3G。如果日志中出现这样的提示或者模拟交易因此提示而失败的话，请优化策略，具体的参考常见Bug或者警告解决方法。

强烈建议模拟盘使用真实价格成交, 即调用 set_option('use_real_price', True). 更多细节

模拟交易替换代码参考教程

替换代码时，版本模拟交易和回测的版本需要对应，python2(python3)版本的模拟交易不能使用python3(python2)版本的回测替换


模拟交易和回测的差别
比较的前提是策略、起始资金、时间区间及频率等完全一致。模拟交易现在和回测还是有些微小的差别, 具体原因如下:

策略运行的环境如果不相同的话，可能导致不同，例如Python2和Python3，聚宽官网和一创聚宽;
策略中有随机因素，例如：使用随机数、不稳定的排序、遍历 dict中的元素等；
回测的策略使用了未来数据，例如开盘前获取当天的收盘价、财务数据、技术指标等；
替换代码：回测不支持替换代码，模拟交易中间替换了代码，回测使用替换后的代码，导致策略不一致；
暂停及重启策略：回测中不支持暂停，模拟交易可以暂停策略，暂停期间策略不运行；

期货交割日
期货持仓到交割日，没有手动交割，系统会以当天结算价平仓, 没有手续费, 不会有交易记录.


还券细则
T+1, 当日融的券当日不能还
还券时要扣除利息
直接还券时, 可以使用当日买入的券还(不受T+1限制), 且优先使用当日买入的券还
投资组合优化器
投资组合优化是指应用概率论与数理统计、最优化方法以及线性代数等相关数学理论方法，根据既定目标收益和风险容许程度（例如最大化收益，最小化风险，风险平价等），将投资重新组合，分散风险的过程，它体现了投资者的意愿和投资者所受到的约束，即在一定风险水平下收益最大化或一定收益水平下的风险最小化。

投资组合管理者在设定了投资收益预期、风险预算、相关约束和风险模型之后， 依托优化器的快速计算优势，得到资产配置最优化结果。

由于不同的约束条件、目标函数，会形成不同的优化器，优化器的处理结果依赖用户输入的相关信息，因此投资者对收益率的预期和风险模型本身估计的准确性，都会影响最终的分析结果，再考虑到交易成本等各类因素的影响，所以从用户使用上而言， 没有绝对意义上最好的优化器。对于资产组合优化问题， 我们可以通过使用优化器，进行一个较长时间的回测，测试整个投资过程，在所有组合输入一致的情况下通过策略的绩效对比来看哪一个优化器有更好的表现， 或者更符合自己的需求。

组合优化器支持对股票、基金进行投资优化，支持如下优化模型：

MinVariance - 组合风险最小化（均值-方差优化）
MaxProfit - 组合收益最大化
MaxSharpeRatio - 组合夏普比率最大化
MinTrackingError - 追踪误差最小化
RiskParity - 风险平价
MaxScore - 组合标的打分最大化
MinScore - 组合标的打分最小化
MaxFactorValue - 因子值最大化
MinFactorValue - 因子值最小化
自定义约束条件的优化模型
对使用优化器的投资组合管理者来说，只需根据收益预期、风险预算，选择恰当的优化模型，并设定相关的约束限制条件。优化器程序可以基于选定的优化模型，输出优化后的投资权重调整建议。我们会对投资组合优化器的进行持续创新与改进。

示例
下面选出上证50成分股的一部分与选定的ETF基金进行组合构成股票池，设定不同的投资组合优化约束条件，并进行回测，测试投资组合优化器对整个投资的影响。

模型1：等权重配置
enter image description here

模型2：组合风险平价；股票的总权重限制为0到90%，ETF的总权重限制为0到10%；每只标的权重不超过10%
enter image description here

模型3：组合风险最小化（最小化组合方差）；组合总权重限制为90%到100%；组合年化收益率目标下限为10%
enter image description here

模型4：'人气指标5日均值'最大化；组合年化收益率目标下限为10%；每只标的权重不超过20%
enter image description here

模型5：组合夏普比率最大化；每只标的权重不超过10%
enter image description here

回测代码如下, 优化函数API详情见 portfolio_optimizer - 投资组合优化：

# 导入函数库
import pandas as pd
from jqdata import *
from jqfactor import Factor
from jqlib.optimizer import *

# 初始化函数，设定基准等等
def initialize(context):
    # 设定沪深300作为基准
    set_benchmark('000300.XSHG')
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)

    # 过滤掉order系列API产生的比error级别低的log
    # log.set_level('order', 'error')

    ### 股票相关设定 ###
    # 股票类每笔交易时的手续费是：买入时佣金万分之三，卖出时佣金万分之三加千分之一印花税, 每笔交易佣金最低扣5块钱
    set_order_cost(OrderCost(close_tax=0.001, open_commission=0.0003, close_commission=0.0003,
                            min_commission=5), type='stock')

    # 优化器设置
    g.optimizer = 2 #设定使用的优化模型
    optimize_model = {
                        1:"模型1：等权重配置",
                        2:"模型2：组合风险平价；股票的总权重限制为0到90%，ETF的总权重限制为0到10%；每只标的权重不超过10%",
                        3:"模型3：组合风险最小化（最小化组合方差）；组合总权重限制为90%到100%；组合年化收益率目标下限为10%",
                        4:"模型4：'人气指标5日均值'最大化；组合年化收益率目标下限为10%；每只标的权重不超过20%",
                        5:"模型5：组合夏普比率最大化；每只标的权重不超过10%"
                      }
    print("优化%s"%(optimize_model[g.optimizer]))

    ## 运行函数（reference_security为运行时间的参考标的；传入的标的只做种类区分，因此传入'000300.XSHG'或'510300.XSHG'是一样的）
      # 开盘前运行
    run_monthly(before_market_open, monthday=1, time='9:00', reference_security='000300.XSHG')
      # 开盘运行
    run_monthly(market_open, monthday=1, time='9:30', reference_security='000300.XSHG')

## 开盘前运行函数
def before_market_open(context):
    print('调仓日期：%s'%context.current_dt.date())

    # 选出上证50成分股的一部分与选定的ETF基金进行组合,构成股票池。
    etf = [
        '159902.XSHE',
        '159903.XSHE',
        '510050.XSHG',
        '510880.XSHG',
        '510440.XSHG',
        ]
    g.buy_list = list(get_index_stocks('000016.XSHG')[-15:]) + etf

## 开盘时运行函数
def market_open(context):
    # 将不在股票池中的股票卖出
    sell_list = set(context.portfolio.positions.keys()) - set(g.buy_list)
    for stock in sell_list:
        order_target_value(stock, 0)

    # 组合优化模型
    if g.optimizer == 1:
        # 模型1：等权重配置
        optimized_weight = pd.Series(data=[1.0/len(g.buy_list)]*len(g.buy_list),
                                    index=g.buy_list)
    elif g.optimizer == 2:
        # 模型2：组合风险平价；股票的总权重限制为0到90%，ETF的总权重限制为0到10%；每只标的权重不超过10%
        optimized_weight = portfolio_optimizer(date=context.previous_date,
                                    securities = g.buy_list,
                                    target = RiskParity(count=250, risk_budget=None),# risk_budget 为 None默认为每只股票贡献相等
                                    constraints = [MarketConstraint('stock', low=0.0, high=0.9),
                                                  MarketConstraint('etf', low=0.0, high=0.1)],
                                    bounds=[Bound(0, 0.1)],
                                    default_port_weight_range=[0., 1.0],
                                    ftol=1e-09,
                                    return_none_if_fail=True)
    elif g.optimizer == 3:
        # 模型3：组合风险最小化（最小化组合方差）；组合总权重限制为90%到100%；组合年化收益率目标下限为10%
        optimized_weight = portfolio_optimizer(date=context.previous_date,
                                    securities = g.buy_list,
                                    target = MinVariance(count=250),
                                    constraints = [WeightConstraint(low=0.9, high=1.0),
                                                   AnnualProfitConstraint(limit=0.1, count=250)],
                                    bounds=[],
                                    default_port_weight_range=[0., 1.0],
                                    ftol=1e-09,
                                    return_none_if_fail=True)
    elif g.optimizer == 4:
        # 模型4：组合标的因子值最大化

        # 定义因子：人气指标5日均值
        class AR(Factor):
            name = 'ar'
            # 每天获取过去五日的数据
            max_window = 5
            # 获取的数据是人气指标
            dependencies = ['AR']
            def calc(self, data):
                return data['AR'].mean()
        # 模型4：'人气指标5日均值'最大化；组合年化收益率目标下限为10%；每只标的权重不超过20%
        optimized_weight = portfolio_optimizer(date=context.previous_date,
                                    securities = g.buy_list,
                                    target = MaxFactorValue(factor=AR, count=1),
                                    constraints = [AnnualProfitConstraint(limit=0.2, count=250)],
                                    bounds=[Bound(0, 0.2)],
                                    default_port_weight_range=[0., 1.0],
                                    ftol=1e-09,
                                    return_none_if_fail=True)
    elif g.optimizer == 5:
        # 模型5：组合夏普比率最大化；每只标的权重不超过10%
        optimized_weight = portfolio_optimizer(date=context.previous_date,
                                    securities = g.buy_list,
                                    target = MaxSharpeRatio(rf=0.0,weight_sum_equal=0.5, count=250),#无风险利率为0，最大化夏普比率需要约束组合权重的和为0.5
                                    constraints = [],
                                    bounds=[Bound(0, 0.1)],
                                    default_port_weight_range=[0., 1.0],
                                    ftol=1e-09,
                                    return_none_if_fail=True)

    # 查看优化结果
    print(optimized_weight)

    # 优化失败，给予警告
    if type(optimized_weight) == type(None):
        print('警告：组合优化失败')
    # 按优化结果，执行调仓操作
    else:
        total_value = context.portfolio.total_value # 获取总资产
        for stock in optimized_weight.keys():
            value = total_value * optimized_weight[stock] # 确定每个标的的权重
            order_target_value(stock, value) # 调整标的至目标权重
策略程序架构♠

名称 描述
initialize 初始化函数
run_daily/run_weekly/run_monthly 定时运行策略(可选)
handle_data 运行策略(可选)
on_event 事件回调(可选)
before_trading_start 开盘前运行策略(可选)
after_trading_end 收盘后运行策略(可选)
on_strategy_end 策略运行结束时调用(可选)
process_initialize 每次程序启动时运行函数(可选)
after_code_changed 模拟交易更换代码后运行函数(可选)
unschedule_all 取消所有定时运行(可选)

策略API介绍

注意事项
【取数据函数】【其它函数】目录中带有"♠" 标识的API是 "回测环境/模拟"专用的API，不能在研究模块中调用。整个 【jqdata 模块】在研究环境与回测环境下都可以使用.
所有价格单位是元
时间表示:
所有时间都是北京时间, 时区:UTC+8
所有时间都是datetime.datetime对象
每个交易日结束时自动撤销所有未完成订单， 例如A股是在17:00之后。
下文中提到 Context, SecurityUnitData, Portfolio, Position, Order 对象都是只读的, 尝试修改他们会报错或者无效.
没有python基础的同学请注意, 有的函数的定义中, 某些参数是有值的, 这个值是参数的默认值, 这个参数是可选的, 可以不传.
回测和模拟中，每日下单的最大数量为10000笔
如需使用策略组合或分仓操作，请看策略组合操作.


策略设置函数
名称 描述
set_benchmark 设置基准
set_order_cost 设置佣金/印花税
set_slippage 设置滑点
use_real_price 设置动态复权(真实价格)模式，建议开启
order_volume_ratio 设置成交量比例
match_with_order_book 设置是否开启盘口撮合模式
set_universe(history专用) 设定股票池
set_commission(已废弃) 设定费率
disable_cache 关闭缓存
实验性设置项 实验性设置项
avoid_future_data 设置是否开启避免未来数据模式
数据获取函数
小提示：

在日级策略中可以获取分钟级K线数据，反之亦然；
取多支标的的数据时，不要获取交易时段不同的标的（例如：不同交易时间的期货标的），否则会报错；
天、分钟、tick行情里成交量单位是股，复权数据中成交量也是复权后的成交量；
更多数据，请访问数据页面查看。
聚宽目前提供哪些数据及数据更新频率
获取频率非一天或者非一分钟的数据，请使用get_bars.
获取数据参考教程：
聚宽新手指南-获取数据教程
数据相关教程
Query及查询财务数据的简单教程
数据常见疑问汇总
数据获取问题快问快答
JQData安装的问题
常用数据获取及计算系列
外部数据获取及分享
【集合贴】数据相关
【API解析】有关数据获取方法
【API解析】get_bars 定义和逻辑
名称 描述
get_price 获取历史数据，可查询多个标的多个数据字段，返回数据格式为 DataFrame
history 获取历史数据，可查询多个标的单个数据字段，返回数据格式为 DataFrame 或 Dict(字典)
attribute_history 获取历史数据，可查询单个标的多个数据字段，返回数据格式为 DataFrame 或 Dict(字典)
get_bars 获取历史数据(包含快照数据)，可查询单个或多个标的多个数据字段，返回数据格式为 numpy.ndarray或DataFrame
get_current_tick♠ 获取最新的 tick 数据
get_ticks 获取股票、期货、50ETF期权、股票指数及场内基金的 tick 数据
get_current_data ♠ 获取当前时间数据
get_extras 获取基金单位/累计净值，期货结算价/持仓量等
get_all_factors 获取聚宽因子库中所有因子的信息
get_factor_values 质量、基础、情绪、成长、风险、每股等数百个因子数据
get_factor_kanban_values 获取因子看板列表数据
get_fundamentals 查询财务数据
get_fundamentals_continuously 查询多日的财务数据
finance.run_query 深沪港通股东信息等数据
macro.run_query 获取聚宽宏观经济数据
get_billboard_list 获取龙虎榜数据
get_locked_shares 获取限售解禁数据
get_index_stocks 获取指数成份股
get_index_weights 获取指数成分股权重
get_industry_stocks 获取行业成份股
get_concept_stocks 获取概念成份股
get_industries 获取行业列表
get_concepts 获取概念列表
get_all_securities 获取所有标的信息
get_security_info 获取单个标的信息
get_industry 查询股票所属行业
get_all_trade_days 获取所有交易日
get_trade_days 获取指定范围交易日
get_money_flow 获取资金流信息
get_concept 获取股票所属概念板块
get_call_auction 获取指定时间区间内集合竞价时的 tick 数据
get_trade_day 根据标的获取指定时刻标的对应的交易日
get_history_fundamentals 获取多个季度/年度的历史财务数据
get_valuation 获取多个标的在指定交易日范围内的市值表数据
jqlib

名称 描述
alpha101 Alpha 101 因子
alpha191 Alpha 191 因子
technical_analysis 技术分析指标
数据处理函数
名称 描述
neutralize 中性化
winsorize 去极值
winsorize_med 中位数去极值
standardlize 标准化(z-score)
组合优化函数
portfolio_optimizer - 投资组合优化
portfolio_optimizer(date, securities, target, constraints, bounds=[Bound(0.0, 1.0)], default_port_weight_range=[0.0, 1.0], ftol=1e-9, return_none_if_fail=True)
优化函数, 用于计算在某些约束条件下的最优组合权重

参数
date: 优化发生的日期，请注意未来函数
securities: 股票代码列表
target: 优化目标函数，只能选择一个，目标函数详见下方
constraints: 限制函数，用以对组合总权重进行限制，可设置一个或多个相同/不同类别的函数，限制函数详见下方
bounds: 边界函数，用以对组合中单标的权重进行限制，可设置一个或多个相同/不同类别的函数，边界函数详见下方。如果不填，默认为 Bound(0., 1.)；如果有多个 bound，则一只股票的权重下限取所有 Bound 的最大值，上限取所有 Bound 的最小值
default_port_weight_range: 长度为2的列表，默认的组合权重之和的范围，默认值为 [0.0, 1.0]。如果限制函数(constraints) 中没有 WeightConstraint 或 WeightEqualConstraint 限制，则会添加 WeightConstraint(low=default_port_weight_range[0], high=default_port_weight_range[1]) 到 constraints列表中。
ftol: 默认为 1e-9，优化函数触发结束的函数值。当求解结果精度不够时可以适当降低 ftol 的值，当求解时间过长时可以适当提高 ftol 值
return_none_if_fail: 默认为 True，如果优化失败，当 return_none_if_fail 为 True 时返回 None，为 False 时返回全为 0 的组合权重
相关参数
参数名称 描述
目标函数(target) 优化目标函数，只能选择一个
限制函数(constraints) 用以对组合总权重进行限制，可设置一个或多个相同/不同类别的函数
边界函数(bounds) 用以对组合中单标的权重进行限制，可设置一个或多个相同/不同类别的函数
示例代码 给了五个应用示例，修改参数即可生效
交易函数
提示

所有下单函数可以在 handle_data中 与 定时运行函数 的 time 参数为 "every_bar"或具体的时间点（例如:time='10:00'）时使用
创建订单失败（返回None）的可能原因： 股票停牌；标的代码错误、已退市、未上市；账户错误(如给股票下单,指定pindex为期货账户)；调整下单手数为0；股票下空单等
了解更多:关于下单函数的说明
名称 描述
order 按股数下单
order_target 目标股数下单
order_value 按价值下单
order_target_value 目标价值下单
cancel_order 撤单
get_open_orders 获取未完成订单
get_orders 获取订单信息
get_trades 获取成交信息
inout_cash 账户出入金
batch_submit_orders 篮子下单
batch_cancel_orders 篮子撤单

对象♠
名称 描述
g 全局变量对象
Context 策略信息总览，包含账户、时间等信息
SubPortfolio 子账户信息
Portfolio 总账户信息
Position 持仓标的信息
SecurityUnitData data对象
tick 对象 tick 对象
Trade对象 订单的一次交易记录,一个订单可能分多次交易.
Order对象 买卖订单信息
OrderStatus 订单状态
OrderStyle 下单方式
Event 事件对象
其他函数
名称 描述
record ♠ 画图函数
send_message ♠ 发送自定义消息
log 日志log信息
write_file 将回测或模拟交易数据写入到投资研究文件中
read_file 在回测或者模拟交易中读取您研究中文件
自定义python库 自定义私人的Python库文件
create_backtest 通过一个策略ID从研究中创建回测
get_backtest 研究中获取回测与模拟交易信息
normalize_code 股票代码格式转换
enable_profile ♠ 性能分析
策略组合操作

名称 描述
set_subportfolios 初始化策略子账户 subportfolios
SubPortfolio 子账户信息
transfer_cash 账户间转移资金
Tick 级策略专用函数

Tick级回测模拟需要权限才可以开通：立即加入会员获取tick权限或者使用积分兑换tick权限

注意Tick级回测必须使用真实价格模式，设置方式详见设置真实价格模式

股票部分：支持 2017-01-01 至今的tick数据，提供买五卖五数据，每3秒一次快照
期货部分：支持 2010-01-01 至今的tick数据，提供买一卖一数据，每0.5秒一次快照
场内基金：支持 2019-01-01 至今的tick数据，提供买五卖五盘口数据，每3秒一次快照
指数：支持 2017-01-01 至今的tick数据，每3秒一次快照
Tick级专用API
名称 描述
handle_tick 策略运行
subscribe 订阅标的的 tick 事件
unsubscribe 取消订阅标的的 tick 事件
unsubscribe_all 取消订阅所有 tick 事件
Tick级示例策略
示例1：
# 回测时间段2019-06-03到2019-06-04，运行频率设置为tick
# 初始化
def initialize(context):
    # 获取起始资金
    init_cash = context.portfolio.starting_cash
    # 交易品种为期货
    set_subportfolios([SubPortfolioConfig(cash=init_cash, type='futures')])
    # 定义一个全局变量, 保存要操作的期货
    g.code1 = 'RB1909.XSGE'

    # 08:30运行自定义开盘前运行函数
    run_daily(before_market_open, time='08:30', reference_security='RB9999.XSGE')
    # 15:30运行自定义收盘后运行函数
    run_daily(after_market_open, time='15:30', reference_security='RB9999.XSGE')

# 开盘前运行函数
def before_market_open(context):
    # 订阅要操作的期货
    subscribe(g.code1, 'tick')

# 有tick事件时运行函数
def handle_tick(context, tick):
    # 获取最新的 tick 数据
    tick_data = get_current_tick(g.code1)
    print(tick_data)

# 收盘后运行函数
def after_market_close(context):
    # 取消今天订阅的标的
    unsubscribe_all()
示例2：
# 导入函数库
import jqdata
def initialize(context):
    set_benchmark('000300.XSHG')
    set_option('use_real_price', True)
    set_order_cost(OrderCost(close_tax=0.001, open_commission=0.0003, close_commission=0.0003, min_commission=5), type='stock')

def before_trading_start(context):
    subscribe('000001.XSHE','tick')

def handle_tick(context, tick):
    log.info(tick)

def after_trading_end(context):
    unsubscribe_all()
融资融券专用函数

初始化融资融券账户
初始化的仓位是不允许直接进行融资融券操作的，因为初始默认 subportfolios[0] 中 SubPortfolioConfig 的 type = 'stock'，只允许买卖股票与场内基金等。

因此要进行融资融券，您需要设定 SubPortfolioConfig 的 type = 'stock_margin'，具体方法如下：

def initialize(context):

    ## 设置单个账户
    # 获取初始资金
    init_cash = context.portfolio.starting_cash 
    # 设定账户为融资融券账户，初始资金为 init_cash 变量代表的数值（如不使用设置多账户，默认只有subportfolios[0]一个账户，Portfolio 指向该账户。）
    set_subportfolios([SubPortfolioConfig(cash=init_cash, type='stock_margin')])

    — — — — — — — — — — — — — — — — — — — — — — — — — — — — — — —
    ## 设置多个账户
    # 获取初始资金，并等分为三份
    init_cash = context.portfolio.starting_cash/3
    # 设定subportfolios[0]为 股票和基金仓位，初始资金为 init_cash 变量代表的数值
    # 设定subportfolios[1]为 金融期货仓位，初始资金为 init_cash 变量代表的数值
    # 设定subportfolios[2]为 融资融券账户，初始资金为 init_cash 变量代表的数值
    set_subportfolios([SubPortfolioConfig(cash=init_cash, type='stock'),\
                       SubPortfolioConfig(cash=init_cash, type='index_futures'),\
                       SubPortfolioConfig(cash=init_cash, type='stock_margin')])
融资融券专用API
注意：get_marginsec_stocks和get_margincash_stocks无法获取当前未完结交易日的数据，因为交易所的数据尚未生成。

名称 描述
margincash_interest_rate 设置融资利率
margincash_margin_rate 设置融资保证金比率
marginsec_interest_rate 设置融券利率
marginsec_margin_rate 设置融券保证金比率
margincash_open 融资买入
margincash_close 卖券还款
margincash_direct_refund 直接还款
marginsec_open 融券卖出
marginsec_close 买券还券
marginsec_direct_refund 直接还券
get_margincash_stocks 获取融资标的列表
get_marginsec_stocks 获取融券标的列表
get_mtss 获取融资融券信息
期货策略专用函数

初始化期货账户
初始化的仓位是不允许直接买卖期货的，因为初始默认 subportfolios[0] 中 SubPortfolioConfig 的 type = 'stock'，只允许买卖股票与场内基金等。
因此要买卖期货，您需要设定 SubPortfolioConfig 的 type = 'futures'，具体方法如下：

def initialize(context):

    ## 设置单个账户
    # 获取初始资金
    init_cash = context.portfolio.starting_cash 
    # 设定账户为金融账户，初始资金为 init_cash 变量代表的数值（如不使用设置多账户，默认只有subportfolios[0]一个账户，Portfolio 指向该账户。）
    set_subportfolios([SubPortfolioConfig(cash=init_cash, type='futures')])

    # 设置运行函数的参考(market_open为自定义函数，需要自己实现；参考标的默认为沪深300，需要根据策略自己设置，下面只是举例)
    run_daily(market_open, time='every_bar', reference_security='CU9999.XSGE')
期货信息
名称 描述
主力连续合约 主力连续合约信息
品种指数 品种指数信息
期货注意事项 期货策略注意事项
期货专用API
了解更多:关于下单函数的说明

名称 描述
get_dominant_future 获取主力合约对应的标的
get_future_contracts 期货可交易合约列表
futures_margin_rate 设置期货保证金比例
is_dangerous 期货保证金预警
get_price等 获取期货的行情数据
order 期货按手数下单
order_target 期货目标手数下单
order_value 期货按保证金下单
order_target_value 期货目标保证金下单
归因分析说明
净值分析
收益分析
累计收益：累计收益
对数轴累计收益：对数轴累计收益
日内收益：每天收益的时间序列图
滑点对列净值曲线的影响：通过受滑点影响的每日收益计算出的累计收益，受滑点影响的每日收益 = 每日收益 - 滑点 * 每日换手率 * 2
年度收益：分年度计算的累计收益的终值
月度收益的时间序列：分月度计算的累计收益的终值
月度收益热力图：分月度计算的累计收益的终值
月度收益频次分布图：查看月度收益频次分布
风险指标
滚动beta指标：滚动 6 个月 (21 * 6 个交易日) 和 12 个月 (21 * 12 个交易日) 的 beta
滚动sharpe指标（6个月时间窗口）：滚动 6 个月 (21 * 6 个交易日) 的夏普比率
前五大回撤区间：找到前5大回撤区间
持仓分析
前10大持仓：显示股票前10大持仓
持仓收益：显示股票的持仓收益，其中内部收益率为一项投资可望到达的报酬率。
日交易股数：每日交易股数的绝对值的和
日换手率：(每日交易市值的绝对值的和 / 2) / 当日总市值
Brinson 归因
基准指沪深300指数，采用减法超额，详细见Brinson模型介绍
总超额收益：策略相对于基准获得的额外收益，是下面主动配置收益、标的选择收益以及交互效应收益的汇总。
主动配置收益：主动配置的收益来源于对上涨行业的超配或对下跌行业的低配，是衡量对大类资产强弱走势进行判断的能力。如果大于零则意味着看准了市场大方向，并且高配了好的资产。
标的选择收益：标的选择的收益来源于对行业中表现好的个股的超配或对行业中表现差个股的低配。是对能否选出高于市场基准收益的资产，即在相同资金分配比例下，能否获得更高的收益能力的衡量。如果大于零则意味着拥有高于市场的个股选择能力。
互动收益：在总超额收益中，除去主动配置收益和标的选择收益，也就是超额收益中同时收到主动配置与标的选择影响的部分，就是互动收益。
因子分析
风格分析
Fama-French五因子模型，是将超额收益分为5个因子来解释，具体如下表

因子	因子解释	构造方式	回归系数解释
市场因子(RM)	受市场走势变化造成的不确定性收益率	市场组合收益率减去无风险收益率	当𝛽𝑖>0
，说明在样本期间内，该组合的运行趋势与市场整体运行趋势是一致的，如果大于1，说明该组合可能偏激进型。
规模因子(SMB)	由于上市公司规模不同导致的收益率差异	小市值组合的收益率减去大市值组合的收益率	当𝑠𝑖>0
，说明该组合可能偏好于配置小盘股
估值因子(HML)	由于上市公司账面市值比不同导致的收益率差异	较高账面市值比的公司组合收益率减去减低账面市值比的公司组合收益率	当ℎ𝑖>0
，说明该组合可能偏好于配置账面市值比高的公司，也就是价值型的公司
盈利因子（RMW）	由于盈利水平不同造成的收益率差异	高盈利公司组合收益率减去低盈利公司组合收益率	当𝑟𝑖>0
，说明该组合可能偏好于配置盈利高的公司
投资因子（CMA）	由于投资水平的不同造成的收益率差异	投资率低的公司组合收益率减去投资率高的公司组合收益率	当𝑐𝑖>0
，说明该组合可能偏好于配置投资率较低的公司
风险分析
说明:

风险因子暴露对比中，基准风险因子暴露度为沪深300指数股票池中股票风险暴露的市值加权平均；风险暴露的基准都是 hs300 指数，所以只要时间段是一样的，暴露度就是一样的
风险因子暴露对比中，风险因子暴露差值代表了策略组合的风险敞口暴露，如果风险因子暴露差值接近0，说明策略组合对该风险因子是风险中性的，即策略组合不暴露与这个风险因子；因子暴露差值就是回测的组合和基准（hs300）组合风险暴露的差，因为这10个因子是大家公认的风险因子，而基准组合被认为是无风险的；所以大家都认为和基准（hs300）组合风险暴露的差越趋近于0，风险暴露越低，策略收益的稳定性越好；大于或小于基准组合风险，可以参考相应因子的定义，调整对应的持仓，将风险暴露趋近于基准组合风险暴露
收益详情中，Backtest为回测的收益;
收益详情中，国家因子是在巴若风险模型中，横截面线性回归的截距项（一共有10个风格因子，11个行业因子(jq_l1)，和1个国家因子）;国家因子的因子值为常数1，因子收益(线性回归的系数)代表了大盘整体的收益;
收益详情中，特殊收益是指收益中剔除已知可解释收益（因子收益）之外的，其他不可解释的收益；
收益详情中，其他的一些收益的解释，点击页面上的提示即可看到；
对收益分析的最后一部分就是查看该策略在各个方面和基准相比的偏差，通过10个风格因子来判别，具体解释如下表。

因子	解释
市值	捕捉大盘股和小盘股之间的收益差异
非线性市值	描述了无法由规模因子解释的但与规模有限的收益差异，通常代表中盘股
杠杆	描述了高杠杆股票与低杠杆股票之间的收益差异
账面市值比	描述了股票估值高低不同而产生的收益差异，即价值因子
成长	描述了对销售或盈利增长预期不同而产生的收益差异
动量	描述了过去半年到一年里相对强势的股票与弱势股票之间的差异
盈利能力	描述了由盈利收益导致的收益差异
贝塔	表征了股票相对于市场的波动敏感程度
残差波动率	解释了剥离了市场风险后的波动率高低产生的收益率差异
流动性	解释了由股票相对的交易活跃度不同而产生的收益率差异。
策略示例
均线策略
当价格高于5日均线平均价格1.05时买入，当价格低于5日平均价格0.95时卖出。

# 导入聚宽函数库
import jqdata

# 初始化函数，设定要操作的股票、基准等等
def initialize(context):
    # 定义一个全局变量, 保存要操作的股票
    # 000001(股票:平安银行)
    g.security = '000001.XSHE'
    # 设定沪深300作为基准
    set_benchmark('000300.XSHG')
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)

# 每个单位时间(如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次)调用一次
def handle_data(context, data):
    security = g.security
    # 获取股票的收盘价
    close_data = attribute_history(security, 5, '1d', ['close'])
    # 取得过去五天的平均价格
    MA5 = close_data['close'].mean()
    # 取得上一时间点价格
    current_price = close_data['close'][-1]
    # 取得当前的现金
    cash = context.portfolio.available_cash

    # 如果上一时间点价格高出五天平均价5%, 则全仓买入
    if (current_price > 1.05*MA5) and (cash>0):
        # 用所有 cash 买入股票
        order_value(security, cash)
        # 记录这次买入
        log.info("Buying %s" % (security))
    # 如果上一时间点价格低于五天平均价, 则空仓卖出
    elif current_price < 0.95*MA5 and context.portfolio.positions[security].closeable_amount > 0:
        # 卖出所有股票,使这只股票的最终持有量为0
        order_target(security, 0)
        # 记录这次卖出
        log.info("Selling %s" % (security))
    # 画出上一时间点价格
    record(stock_price=current_price)
多股票持仓示例
这是一个较简单的多股票操作示例，当价格高于三天平均价1.005则买入100股，当价格小于三天平均价0.995则卖出。

# 导入聚宽函数库
import jqdata

def initialize(context):
    # 初始化此策略
    # 设置我们要操作的股票池
    g.stocks = ['000001.XSHE','000002.XSHE','000004.XSHE','000005.XSHE']
    # 设定沪深300作为基准
    set_benchmark('000300.XSHG')
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)

# 每个单位时间(如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次)调用一次
def handle_data(context, data):
    # 循环每只股票
    for security in g.stocks:
        # 得到股票之前3天的平均价
        vwap = data[security].vwap(3)
        # 得到上一时间点股票收盘价
        price = data[security].close
        # 得到当前资金余额
        cash = context.portfolio.available_cash

        # 如果上一时间点价格小于三天平均价*0.995，并且持有该股票，卖出
        if price < vwap * 0.995 and context.portfolio.positions[security].closeable_amount > 0:
            # 下入卖出单
            order(security,-100)
            # 记录这次卖出
            log.info("Selling %s" % (security))
        # 如果上一时间点价格大于三天平均价*1.005，并且有现金余额，买入
        elif price > vwap * 1.005 and cash > 0:
            # 下入买入单
            order(security,100)
            # 记录这次买入
            log.info("Buying %s" % (security))
多股票追涨策略
当股票在当日收盘30分钟内涨幅到达9.5%~9.9%时间段的时候，我们进行买入，在第二天开盘卖出。注意：请按照分钟进行回测该策略。

# 导入聚宽函数库
import jqdata

# 初始化程序, 整个回测只运行一次
def initialize(context):
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)

    # 每天买入股票数量
    g.daily_buy_count  = 5

    # 设置我们要操作的股票池, 这里我们操作多只股票，下列股票选自计算机信息技术相关板块
    g.stocks = get_industry_stocks('I64') + get_industry_stocks('I65')

    # 防止板块之间重复包含某只股票, 排除掉重复的, g.stocks 现在是一个集合(set)
    g.stocks = set(g.stocks)

    # 让每天早上开盘时执行 morning_sell_all
    run_daily(morning_sell_all, '09:30')

def morning_sell_all(context):
    # 将目前所有的股票卖出
    for security in context.portfolio.positions:
        # 全部卖出
        order_target(security, 0)
        # 记录这次卖出
        log.info("Selling %s" % (security))

def before_trading_start(context):
    # 今天已经买入的股票
    g.today_bought_stocks = set()

    # 得到所有股票昨日收盘价, 每天只需要取一次, 所以放在 before_trading_start 中
    g.last_df = history(1,'1d','close',g.stocks)

# 在每分钟的第一秒运行, data 是上一分钟的切片数据
def handle_data(context, data):

    # 判断是否在当日最后的2小时，我们只追涨最后2小时满足追涨条件的股票
    if context.current_dt.hour < 13:
        return

    # 每天只买这么多个
    if len(g.today_bought_stocks) >= g.daily_buy_count:
        return

    # 只遍历今天还没有买入的股票
    for security in (g.stocks - g.today_bought_stocks):

        # 得到当前价格
        price = data[security].close

        # 获取这只股票昨天收盘价
        last_close = g.last_df[security][0]

        # 如果上一时间点价格已经涨了9.5%~9.9%
        # 今天的涨停价格区间大于1元，今天没有买入该支股票
        if price/last_close > 1.095 \
                and price/last_close < 1.099 \
                and data[security].high_limit - last_close >= 1.0:

            # 得到当前资金余额
            cash = context.portfolio.available_cash

            # 计算今天还需要买入的股票数量
            need_count = g.daily_buy_count - len(g.today_bought_stocks)

            # 把现金分成几份,
            buy_cash = context.portfolio.available_cash / need_count

            # 买入这么多现金的股票
            order_value(security, buy_cash)

            # 放入今日已买股票的集合
            g.today_bought_stocks.add(security)

            # 记录这次买入
            log.info("Buying %s" % (security))

            # 买够5个之后就不买了
            if len(g.today_bought_stocks) >= g.daily_buy_count:
                break
万圣节效应策略
股市投资中的“万圣节效应”是指在北半球的冬季(11月至4月份)，股市回报通常明显高於夏季(5月至10月份)。这里我们选取了中国蓝筹股，采用10月15日后买入，5月15日后卖出的简单策略进行示例。

# 导入聚宽函数库
import jqdata

# 初始化此策略
def initialize(context):
    # 开启动态复权模式(真实价格)
    set_option('use_real_price', True)

    # 设置我们要操作的股票池，这里我们选择蓝筹股
    g.stocks = ['000001.XSHE','600000.XSHG','600019.XSHG','600028.XSHG','600030.XSHG','600036.XSHG','600519.XSHG','601398.XSHG','601857.XSHG','601988.XSHG']

# 每个单位时间(如果按天回测,则每天调用一次,如果按分钟,则每分钟调用一次)调用一次
def handle_data(context, data):
    # 得到每只股票可以花费的现金，这里我们使用总现金股票数数量
    cash = context.portfolio.available_cash / len(g.stocks)
    # 获取数据
    hist = history(1,'1d','close',g.stocks)
    # 循环股票池
    for security in g.stocks:
        # 得到当前时间
        today = context.current_dt
        # 得到该股票上一时间点价格
        current_price = hist[security][0]
        # 如果当前为10月且日期大于15号，并且现金大于上一时间点价格，并且当前该股票空仓
        if today.month == 10 and today.day > 15 and cash > current_price and context.portfolio.positions[security].closeable_amount == 0:
            order_value(security, cash)
            # 记录这次买入
            log.info("Buying %s" % (security))
        # 如果当前为5月且日期大于15号，并且当前有该股票持仓，则卖出
        elif today.month == 5 and today.day > 15 and context.portfolio.positions[security].closeable_amount > 0:
            # 全部卖出
            order_target(security, 0)
            # 记录这次卖出
            log.info("Selling %s" % (security))
