❯ 当前我的状态机及可能有很多问题，你能帮我检查一下嘛                                                                                                                         
                                                     
● 我来仔细检查巡航状态机的逻辑。                                                                                                                                             
   
● Read 1 file (ctrl+o to expand)                                                                                                                                             
                  
● 好的，我来仔细分析状态机的问题。                                                                                                                                           
                               
  ---                                                                                                                                                                        
  发现的问题                                                                                                                                                                 
                                                                                                                                                                             
  问题1：CRUISE_WAIT_PRESET 中 queryElapsedMs 计数逻辑错误                                                                                                                   
                  
  // 每次循环都累加
  fsm->queryElapsedMs += CRUISE_LOOP_INTERVAL_MS;

  if(fsm->queryPending)
  {
      fsm->queryWaitMs += CRUISE_LOOP_INTERVAL_MS;
      // ...
  }
  else if(fsm->queryElapsedMs >= CRUISE_QUERY_INTERVAL_MS)
  {
      fsm->queryElapsedMs = 0;  // 发送查询后清零
      // ...
  }

  问题：queryElapsedMs 在 queryPending == true 期间仍然继续累加，但此时走的是 if(fsm->queryPending) 分支，不会触发 else if。等 queryPending 清零后，queryElapsedMs
  可能已经远超 CRUISE_QUERY_INTERVAL_MS，导致下一次查询立即发出，没有等待间隔。

  修复：发送查询时同时重置 queryElapsedMs，或者在 queryPending 期间暂停累加。

  ---
  问题2：stableElapsedMs 累加步长用错

  if(!fsm->ptStable)
  {
      fsm->ptStable = ZS_TRUE;
      fsm->stableElapsedMs = CRUISE_QUERY_INTERVAL_MS;  // 初始值200ms
  }
  else
  {
      fsm->stableElapsedMs += CRUISE_QUERY_INTERVAL_MS;  // 每次加200ms
  }

  问题：稳定判断的实际时间间隔是 queryWaitMs（100ms）触发后才执行，即每次稳定检查的真实间隔约为 CRUISE_QUERY_INTERVAL_MS(200ms) + queryWaitMs(100ms) = 300ms，但
  stableElapsedMs 每次只加 200ms，导致实际稳定等待时间比预期的 CRUISE_STABLE_HOLD_MS(400ms) 更长。

  ---
  问题3：CRUISE_WAIT_PRESET 无超时保护

  如果云台因故障、堵转等原因永远到不了目标预置点，状态机会永远卡在 CRUISE_WAIT_PRESET，巡航永远不会继续。

  建议：增加一个总超时计数器，超过阈值（如30秒）后强制跳转到下一个预置点。

  ---
  问题4：CRUISE_CONFIG_UPDATE 处理有竞态风险

  在 zs_Ptz_Cruise_Thread 中：

  if(g_curCruiseStatus == CRUISE_CONFIG_UPDATE)
  {
      g_curCruiseStatus = CRUISE_START;   // 先改状态
      fsm.state = CRUISE_STOP;
      // ...
  }

  handle_cruise_state(&fsm);  // 紧接着执行，fsm.state=CRUISE_STOP，g_curCruiseStatus=CRUISE_START

  这个逻辑本身没问题，但 g_curCruiseStatus 是全局变量，被主线程和巡航线程同时读写，没有任何互斥保护（无 mutex、无 atomic）。在多核 ARM64 上存在可见性问题。

  ---
  问题5：CRUISE_RUNNING 状态下 ZS_PTZ_CallOnePreset 内部会清除 g_bWatchCruiseRunning

  // CRUISE_RUNNING 分支
  ZS_BOOL savedWatchCruiseFlag = g_bWatchCruiseRunning;
  g_bWatchCruiseRunning = ZS_FALSE;
  ZS_PTZ_CallOnePreset(...);          // 内部会执行: g_bWatchCruiseRunning = ZS_FALSE
  g_bWatchCruiseRunning = savedWatchCruiseFlag;  // 恢复

  看 ZS_PTZ_CallOnePreset（第1211-1215行）：

  ZS_INT32 ZS_PTZ_CallOnePreset(...)
  {
      if (g_bWatchCruiseRunning && g_curCruiseStatus != CRUISE_STOP) {
          ZS_PTZ_StopOneCruise(...);   // 此时 g_bWatchCruiseRunning 已被临时置 FALSE，不会进这里
      }
      g_bWatchCruiseRunning = ZS_FALSE;  // ← 这里会把 savedWatchCruiseFlag 的恢复覆盖掉！

  问题：ZS_PTZ_CallOnePreset 末尾无条件执行 g_bWatchCruiseRunning = ZS_FALSE，在 CRUISE_RUNNING 恢复 savedWatchCruiseFlag
  之后，这行代码已经执行完了，所以恢复是有效的。但如果未来有人调整执行顺序，这个隐式依赖很容易出错。逻辑上 ZS_PTZ_CallOnePreset 不应该无条件清除这个标记。

  ---
  问题6：CRUISE_WAIT_STAY_TIME 使用 mktime 比较时间有副作用

  if(mktime(&cur_time) - mktime(&fsm->startTime) >= fsm->cruise_info[...].stayTime)

  mktime 会修改传入的 struct tm（归一化字段），对 fsm->startTime 反复调用 mktime 会累积修改，虽然通常无害，但属于不规范用法。

  建议：改用 time_t 存储 startTime，直接做减法。

  ---
  总结


  ┌────────┬─────────────────────────────────────────┬──────────────┐
  │ 优先级 │                  问题                   │     影响     │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 高     │ 无超时保护（问题3）                     │ 巡航卡死     │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 高     │ 全局状态无互斥（问题4）                 │ 偶发竞态     │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 中     │ queryElapsedMs 累加错误（问题1）        │ 查询间隔不准 │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 中     │ stableElapsedMs 步长偏差（问题2）       │ 到位判断偏慢 │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 低     │ g_bWatchCruiseRunning 隐式依赖（问题5） │ 维护风险     │
  ├────────┼─────────────────────────────────────────┼──────────────┤
  │ 低     │ mktime 修改 startTime（问题6）          │ 不规范       │
  └────────┴─────────────────────────────────────────┴──────────────┘

  需要我帮你逐一修复这些问题吗？
