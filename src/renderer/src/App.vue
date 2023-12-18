<script setup>
import 'element-plus/es/components/message/style/css'

import { ref, watch, reactive } from 'vue'
import { CircleCloseFilled } from '@element-plus/icons-vue'
import { ElNotification, ElMessage } from 'element-plus'
const drawer = ref(false)




const el_tag_size = ref("10px");

class Record { // 记录每个进程的行踪
  process; // 进程
  timeMap; // 对应时间每个进程发生的变化
  constructor(p) {
    this.process = p;
    this.timeMap = new Map();
  }
}
const drawerRecord = ref(false);
// 根据进程获得记录
const getRecordByProcess = (pro) => {
  let res = pro_record.value.get(pro);
  if (res === undefined) return null;
  return  res;
}
// const IORecord = ref(new Map());


const getClass = (change) => {
  if (change === "就绪 => 运行") return "rr";
  else if (change === "运行 => 阻塞") return "rb";
  else if (change === "阻塞 => 就绪") return "br";
  else if (change === "创建 => 就绪") return "cr";
  else if (change === "运行 => 终止") return "re";
  else if (change === "运行 => 就绪") return  "rr1";
  return "no";
}

const getClass1 = (item) => {
  if (item.IoReq === null || item.IoReq === undefined){
    if (item.highlight) return "withHighlight";
    else return "";
  } else {
    if (item.IoReq.device === "printer" && item.highlight) return "withHighlight";
    if (item.IoReq.device === "keyboard" && item.highlight) return "withHighlight";
    if (item.IoReq.device === "printer") return "withPrinter";
    if (item.IoReq.device === "keyboard") return "withKeyboard";
  }
}

class PCB {
  //进程名
  name;
  //进程ID
  ID;
  //优先级
  priority;
  //到达时间
  arriveTime;
  //需运行时间
  serviceTime;
  //已用CPU时间
  useCPUTime;
  //进程状态
  status;
  //阻塞原因
  blockReason;
  //等待时间
  waitTime;
  //高亮(界面效果)
  highlight;
  // 唤醒时间
  wakeTime;
  // io请求
  IoReq;
  // 是否是io请求造成的阻塞
  isIO;
  constructor(name, ID, serviceTime, priority, arriveTime) { // 构造器
    this.name = name;
    this.ID = ID;
    this.priority = priority;
    this.arriveTime = arriveTime;
    this.serviceTime = serviceTime;
    if (this.arriveTime === context.value.passTime) {
      this.status = "ready";
    } else {
      this.status = "new";
    }
    this.useCPUTime = 0;
    this.blockReason = "";
    this.waitTime = 0;
    this.highlight = false;
    this.wakeTime = 0;
    this.isIO = false;
    this.IoReq = null;
  }
}

class Device { // 设备
  name;
  number;
  applyTime;
  state = [];
  IORecord;
  constructor(name, number, applyTime) {
    this.name = name;
    this.number = number;
    this.applyTime = applyTime;
    this.state = new Array(number).fill(0);
    this.IORecord = new Map();
  }
}

const printer = reactive(new Device("printer", 5, 7));
const keyboard = reactive(new Device("keyboard", 1, 3));

// 记录与process之间的对应
const pro_record = ref(new Map());

let deviceList = reactive([]);
deviceList.push(printer, keyboard)

class IoRequest {
  device; // 使用的设备名称
  runTime;
  process_id;
  device_id;
  constructor(device, runTime) {
    this.device = device;
    this.runTime = runTime;
  }
}

const formLabelWidth = '140px'

const form = reactive({
  name: '',
  time: 0,
})

// 设备全部申请完触发
const deviceLack = (message) => {
  ElMessage.error(message)
}

// const needRunDevice = ref([]);
const map = new Map();

const doWake1 = (id) => {
  for (let i = 0; i < context.value.blockQueue.length; i++) {
    let item = context.value.blockQueue[i];
    if (item.ID === id) {
      context.value.blockQueue.splice(i, 1);
      context.value.readyQueues[0].push(item);
      // 找到设备，使它的数量增加
      for (let device of deviceList) {
        if (device.name === item.IoReq.device) {
          // needRunDevice.value.push({
          //   "device": device,
          //   "process": item,
          // });
          item.isIO = false;
          map.set(item, device);
          // device.number++;
          // device.state[item.IoReq.device_id] = 0;
        }
      }
    }
  }
}
const dialogFormVisibl2 = ref(false)
const waitDeviceList = ref([]);
const waitDeviceList1 = ref([]);

const deviceArrive = (message) => {
  ElMessage({
    message: message,
    type: 'success',
  })
}

//阻塞当前运行进程
const doBlock1 = () => {
  const runProcess = context.value.runQueue[0];
  if (runProcess.IoReq !== null && runProcess.IoReq !== undefined) {
    if (runProcess.IoReq.runTime !== 0) {
      deviceLack(`进程 ${runProcess.name} 正在使用设备 ${runProcess.IoReq.device}`);
      return;
    }
  }
  const IoReq = new IoRequest(form.name, form.time);
  for (let device of deviceList) {
    if (device.name === IoReq.device) { // 找到设备
      let number = device.number;
      if (context.value.runQueue.length !== 0) {
        let item = context.value.runQueue.shift();
        let record = getRecordByProcess(item);
        if (number <= 0) { // 请求资源失败
          item.status = "block";
          context.value.blockQueue.push(item);
          deviceLack(`${device.name}已经用完了, 进程 ${item.name} 进入阻塞状态，等待设备`); // 触发
          device.IORecord.set(context.value.passTime, `进程 ${item.name} 发送I/O请求，申请设备，此时设备数量为${number}，拒绝此次申请`);
          IoReq.process_id = item.ID;
          item.IoReq = IoReq;
          item.isIO = true;
          // TODO
          if (device.name === "printer") waitDeviceList.value.push(item)
          else waitDeviceList1.value.push(item)

          if (record !== null) {
            const re = {
              "change" : "运行 => 阻塞",
              "content": `进程申请设备 ${device.name} 编号为 ${item.IoReq.device_id} 的设备失败(该设备已被其他进程分配完)，转换为阻塞状态，进入阻塞队列`,
            }
            record.timeMap.set(context.value.passTime, re);
            // record.timeMap.set(context.value.passTime, `运行 => 阻塞: 进程申请设备${device.name}失败(该设备已被其他进程分配完)，转换为阻塞状态，进入阻塞队列`);
          }
        } else { // 等待一段时间自动唤醒
          IoReq.process_id = item.ID;
          item.status = "block";
          context.value.blockQueue.push(item);
          item.wakeTime = item.waitTime + device.applyTime;
          device.number--;
          for (let i = 0; i < device.state.length; i++) {
            if (device.state[i] === 0) {
              device.state[i] = 1;
              IoReq.device_id = i;
              device.IORecord.set(context.value.passTime, `进程 ${item.name} 发送I/O请求，申请设备，此时设备数量为 ${device.number+1} ，同意此次申请，
          分配设备 ${device.name} 中的编号为 ${i} 的设备给该进程`);
              break;
            }
          }
          item.IoReq = IoReq;
          item.serviceTime += item.IoReq.runTime;
          deviceArrive(`${device.name} 已被进程 ${item.name} 占用, 此时个数为 ${device.number}`);
          if (record !== null) {
            const re = {
              "change" : "运行 => 阻塞",
              "content": `进程申请设备 ${device.name} 编号为 ${item.IoReq.device_id} 的设备成功，转换为阻塞状态，进入阻塞队列，等待I/O请求结束`,
            }
            record.timeMap.set(context.value.passTime, re);
            // record.timeMap.set(context.value.passTime, `运行 => 阻塞: 进程申请设备${device.name}成功，转换为阻塞状态，进入阻塞队列，等待I/O请求结束`);
          }
        }
      }
    }
  }

}

const selectedSchedulingAlgorithIndex = ref("");

const initData = [
  {
    name: "空",
    data: [],
  },
  {
    name: "数据一(可用来测试先来先服务)",
    data: [
      { name: "进程-0", ID: 0, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "进程-1", ID: 1, serviceTime: 4, priority: 0, arriveTime: 0 },
      { name: "进程-2", ID: 2, serviceTime: 3, priority: 0, arriveTime: 0 },
      { name: "进程-3", ID: 3, serviceTime: 2, priority: 0, arriveTime: 0 }
    ],
  },
  {
    name: "数据二(可用来测试时间片轮转)",
    data: [
      { name: "进程-0", ID: 0, serviceTime: 6, priority: 0, arriveTime: 0 },
      { name: "进程-1", ID: 1, serviceTime: 7, priority: 0, arriveTime: 0 },
      { name: "进程-2", ID: 2, serviceTime: 8, priority: 0, arriveTime: 0 },
      { name: "进程-3", ID: 3, serviceTime: 9, priority: 0, arriveTime: 0 }
    ],
  },
  {
    name: "数据三(可用来测试优先级)",
    data: [
      { name: "进程-0", ID: 0, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "进程-1", ID: 1, serviceTime: 5, priority: 2, arriveTime: 0 },
      { name: "进程-2", ID: 2, serviceTime: 5, priority: 4, arriveTime: 0 },
      { name: "进程-3", ID: 3, serviceTime: 5, priority: 6, arriveTime: 0 }
    ],
  },
  {
    name: "数据四(可用来测试高响应比优先)",
    data: [
      { name: "进程-0", ID: 0, serviceTime: 8, priority: 0, arriveTime: 0 },
      { name: "进程-1", ID: 1, serviceTime: 7, priority: 0, arriveTime: 3 },
      { name: "进程-2", ID: 2, serviceTime: 3, priority: 0, arriveTime: 5 },
      { name: "进程-3", ID: 3, serviceTime: 2, priority: 0, arriveTime: 7 }
    ],
  },
  {
    name: "数据五(可用来测试多级反馈队列)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 10, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 16, priority: 0, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 8, priority: 0, arriveTime: 0 }
    ],
  },
]


//先来先服务算法对象
const FCFS = {
  schedule: function () {
    if (context.value.runQueue.length !== 0) { return; }
    if (context.value.readyQueues[0].length === 0) { return; }
    let index = context.value.readyQueues[0].findIndex(i => i.arriveTime === Math.min(...context.value.readyQueues[0].map(j => j.arriveTime)));
    // let item = context.value.readyQueues[0].shift();
    let item = context.value.readyQueues[0].splice(index, 1)[0];
    item.status = 'running';
    item.highlight = false;
    let record = getRecordByProcess(item)
    if (record !== null) {
      const re = {
        "change" : "就绪 => 运行",
        "content": `根据调度算法FCFS，选择最先到达的进程 ${item.name} ，该进程由就绪状态转化为运行状态`,
      }
      record.timeMap.set(context.value.passTime, re);
    }
    context.value.runQueue.push(item);
  }
}

//时间片，默认为1
const RR_timeSlice = ref(1);
//时间片轮转算法对象
const RR = {
  //当前已用时间片
  tmpTimeSlice: 0,
  schedule: function () {
    RR.tmpTimeSlice++;
    if (RR.tmpTimeSlice === 1 || context.value.runQueue.length === 0) {
      RR.tmpTimeSlice = 1;
      //放出
      if (context.value.runQueue.length !== 0) {
        let item = context.value.runQueue.shift();
        context.value.readyQueues[0].push(item);
        item.status = 'ready';
        item.highlight = false;
        let record = getRecordByProcess(item)
        if (record !== null) {
          const re = {
            "change" : "运行 => 就绪",
            "content": `根据调度算法RR，进程 ${item.name} 分配的时间片已用完 由运行状态转化为就绪状态`,
          }
          record.timeMap.set(context.value.passTime, re);
        }
      }
      //放入
      if (context.value.readyQueues[0].length !== 0) {
        let item = context.value.readyQueues[0].shift();
        context.value.runQueue.push(item);
        item.status = 'running';
        item.highlight = false;
        let record = getRecordByProcess(item)
        if (record !== null) {
          const re = {
            "change" : "就绪 => 运行",
            "content": `根据调度算法RR，进程 ${item.name} 分配时间片，由就绪状态转化为运行状态`,
          }
          record.timeMap.set(context.value.passTime, re);
        }
      }
    }
    if (RR.tmpTimeSlice === RR_timeSlice.value) {
      RR.tmpTimeSlice = 0;
    }

  }
}
//优先级调度算法对象
const PSA = {
  schedule: function () {
    if (context.value.runQueue.length !== 0) { return; }
    if (context.value.readyQueues[0].length === 0) { return; }
    let flag = 0;
    for (let i = 1; i < context.value.readyQueues[0].length; i++) {
      let a = context.value.readyQueues[0][flag];
      let b = context.value.readyQueues[0][i];
      if (a.priority < b.priority) { flag = i; }
    }
    let item = context.value.readyQueues[0][flag];
    context.value.readyQueues[0].splice(flag, 1);
    item.status = 'running';
    item.highlight = false;
    context.value.runQueue.push(item);
    let record = getRecordByProcess(item)
    if (record !== null) {
      const re = {
        "change" : "就绪 => 运行",
        "content":`根据调度算法PSA，选择优先级最高的 进程${item.name} ，该进程由就绪状态转化为运行状态`,
      }
      record.timeMap.set(context.value.passTime, re);
    }
  }
}
//高响应比优先调度算法对象
const HRRN = {
  schedule: function () {
    if (context.value.runQueue.length !== 0) { return; }
    if (context.value.readyQueues[0].length === 0) { return; }
    let flag = 0;
    for (let i = 1; i < context.value.readyQueues[0].length; i++) {
      let item1 = context.value.readyQueues[0][flag];
      let item2 = context.value.readyQueues[0][i];
      // let rr1 = 1 + (context.value.passTime - 1 - item1.arriveTime) / item1.serviceTime;
      // let rr2 = 1 + (context.value.passTime - 1 - item1.arriveTime) / item2.serviceTime;
      let rr1 = 1 + (item1.waitTime / item1.serviceTime);
      let rr2 = 1 + (item2.waitTime / item2.serviceTime);
      if (rr1 < rr2) {
        flag = i;
      }
    }
    let item = context.value.readyQueues[0][flag];
    item.status = 'running';
    item.highlight = false;
    context.value.readyQueues[0].splice(flag, 1);
    context.value.runQueue.push(item);
    let record = getRecordByProcess(item)
    if (record !== null) {
      const re = {
        "change" : "就绪 => 运行",
        "content":`根据调度算法HRRN，选择响应比最高的进程 ${item.name} ，该进程就绪状态转化为运行状态`,
      }
      record.timeMap.set(context.value.passTime, re);
    }
  },
}

const context = ref({
  recordQueue: [],
  newQueue: [],
  readyQueues: [],
  runQueue: [],
  blockQueue: [],
  finishQueue: [],
  //0未开始,1运行中,2暂停
  globalStatus: 0,
  passTime: 0,
  globalTaskTimer: null,
  schedulingAlgorithm: null,
  processNum: 0,
  clear: function () {
    context.value.recordQueue = [];
    context.value.newQueue = [];
    context.value.readyQueues = [];
    context.value.runQueue = [];
    context.value.blockQueue = [];
    context.value.finishQueue = [];
    context.value.passTime = 0;
    context.value.processNum = 0;
    // context.value.globalTaskTimer = null;
    context.value.schedulingAlgorithm = null;
    selectedSchedulingAlgorithIndex.value = 0;
    RR_timeSlice.value = 1;
    selectedInitProcessDataIndex.value = 0;
    selectedBlockProcessIndex.value = [];
    dialogFormVisible.value = false;
  }
})

const viewDeviceNumber = (device, waitDeviceList1) => {
  if (device.number !== 0 && waitDeviceList1.value.length !== 0) {
    deviceArrive(`${name}有空余设备, 此时个数为${device.number}`);
    let n = Math.min(device.number, waitDeviceList1.value.length);
    let deleteList = [];
    let pro_name = "";
    for (let i = 0; i < n; i++) {
      let item = waitDeviceList1.value[i];
      pro_name = item.name;
      item.wakeTime = item.waitTime + device.applyTime;
      for (let j = 0; j < device.state.length; j++) {
        if (device.state[j] === 0) {
          device.state[j] = 1;
          item.IoReq.device_id = j;
          item.isIO = false; // 抢占到了，取消高亮显示
          break;
        }
      }
      item.serviceTime += item.IoReq.runTime;
      deleteList.push(i);
      device.IORecord.set(context.value.passTime, `进程 ${item.name} 申请设备 ${device.name} 编号为 ${item.IoReq.device_id} 的设备, 此时设备数量为 ${device.number}, 同意这次申请`);
      device.number--;
      let record = getRecordByProcess(item);
      if (record !== null) {
        const re = {
          "change": "无状态变化",
          "content": `等待设备空闲的进程申请设备 ${device.name} 编号为 ${item.IoReq.device_id} 的设备成功`,
        }
        record.timeMap.set(context.value.passTime, re);
      }
      setTimeout(()=> {
        deviceArrive(`${name}已被进程 ${pro_name} 占用, 此时个数为 ${device.number}`);
      }, 500)
    }
    for (let x in deleteList) waitDeviceList1.value.splice(x, 1);
  }
}

//每秒改变当前界面内容
const changeState = () => {
  //全局状态
  context.value.passTime++;
  //更新队列时间
  // context.value.newQueue.forEach(item => { item.waitTime++; })

  deviceList.forEach(device => {
    if (device.name === "keyboard")  viewDeviceNumber(device, waitDeviceList1);
    if (device.name === "printer") viewDeviceNumber(device, waitDeviceList);
  })

  context.value.readyQueues.forEach(q => {
    q.forEach(item => { item.waitTime++; })
  })

  context.value.runQueue.forEach(item => {
    item.useCPUTime++;
    if (map.get(item) !== undefined) {
      if (item.IoReq.runTime - 1 === 0) {
        console.log(item.IoReq);
        map.get(item).number++;
        map.get(item).state[item.IoReq.device_id] = 0;
        map.get(item).IORecord.set(context.value.passTime, `进程 ${item.name} 终止对设备编号为 ${item.IoReq.device_id} 的设备的使用，此时设备数量为 ${map.get(item).number},
         回收设备${map.get(item).name}编号为${item.IoReq.device_id}的设备`);
        deviceArrive(`进程 ${item.name} 使用设备 ${map.get(item).name} 结束`);
        let record = getRecordByProcess(item)
        if (record !== null) {
          const re = {
            "change" : "无状态转换",
            "content": `进程 ${item.name} 使用设备 ${map.get(item).name} 编号为 ${item.IoReq.device_id} 的设备结束`,
          }
          record.timeMap.set(context.value.passTime, re);
        }
        map.delete(item);
        item.IoReq.runTime--;
        item.IoReq = null;
      } else item.IoReq.runTime--;
    }
  })

  context.value.blockQueue.forEach(item => {
    item.waitTime++;
    // console.log(item.waitTime)
    if (item.wakeTime !== 0) {
      if (item.waitTime === item.wakeTime) {
        let record = getRecordByProcess(item)
        if (record !== null) {
          const re = {
            "change" : "阻塞 => 就绪",
            "content":`进程 ${item.name} 申请设备 ${item.IoReq.device} 编号为 ${item.IoReq.device_id} 的设备的I/O请求结束，转换为就绪状态，进入就绪队列`,
          }
          record.timeMap.set(context.value.passTime, re);
        }
        if (map.get(item) !== undefined) {
          map.get(item).IORecord.set(context.value.passTime, `进程 ${item.name} 的I/O请求结束，申请设备 ${map.get(item).name} 编号为 ${item.IoReq.device_id} 的设备成功,
        此时设备数量为${map.get(item).number}，当前设备是${item.IoReq.device}中`);
        }
       doWake1(item.ID);
      }
    }
  })

  //再更新状态
  //run->finish
  if (context.value.runQueue.length > 0) {
    let runItem = context.value.runQueue[0];
    if (runItem.useCPUTime === runItem.serviceTime) {
      context.value.runQueue.shift();
      runItem.status = "finish";
      context.value.finishQueue.push(runItem);
      runItem.highlight = false;
      let record = getRecordByProcess(runItem)
      if (record !== null) {
        const re = {
          "change" : "运行 => 终止",
          "content": "进程任务执行结束，退出运行队列",
        }
        record.timeMap.set(context.value.passTime,  re);
      }
      // runItem.waitTime--;
    }
  }
  //ready->run
  context.value.schedulingAlgorithm.schedule();
  //new->ready
  for (let i = context.value.newQueue.length - 1; i >= 0; i--) {
    let item = context.value.newQueue[i];
    if (context.value.passTime === item.arriveTime) {
      context.value.newQueue.splice(i, 1);
      item.status = "ready";
      context.value.readyQueues[0].push(item);
      item.highlight = false;
      let record = getRecordByProcess(item)
      if (record !== null) {
        const re = {
          "change" : "创建 => 就绪",
          "content":"进程到达系统，转换为就绪状态，进入就绪队列",
        }
        record.timeMap.set(context.value.passTime, re);
      }
    }
  }
}

const doTimingGlobalTask = () => {
  context.value.globalTaskTimer = setTimeout(() => {
    if (context.value.globalStatus != 1) { return; }
    changeState();
    doTimingGlobalTask();
  }, 1000)
}

const selectedInitProcessDataIndex = ref(0);

// globalStatus = 1 自动运行 globalStatus = 2 手动运行
const start = () => {
  context.value.schedulingAlgorithm = schedulingAlgorithms[selectedSchedulingAlgorithIndex.value].sa;
  context.value.readyQueues.push([]);
  context.value.processNum = initData[selectedInitProcessDataIndex.value].data.length;
  initData[selectedInitProcessDataIndex.value].data.forEach(item => {
    let p = new PCB(item.name, item.ID, item.serviceTime, item.priority, item.arriveTime);
    let record = new Record(p); // 记录
    if (p.arriveTime == 0) {
      context.value.readyQueues[0].push(p);
      const re = {
        "change" : "创建 => 就绪",
        "content": "进程已到达就绪队列",
      }
      record.timeMap.set(context.value.passTime, re);
    } else {
      context.value.newQueue.push(p);
      const re = {
        "change" : "无状态转换",
        "content": "进程未到达系统中",
      }
      record.timeMap.set(context.value.passTime, re);
    }
    context.value.recordQueue.push(record);
    pro_record.value.set(p, record);
  })
  context.value.schedulingAlgorithm.schedule();
  if (context.value.globalStatus == 1) {
    doTimingGlobalTask();
  } else {
    // next();
  }
}


const pause = () => {
  context.value.globalStatus = 2;
}
const continuee = () => {
  context.value.globalStatus = 1;
  context.value.globalTaskTimer = setTimeout(() => {
    changeState();
    doTimingGlobalTask();
  }, 1000)
}
const end = () => {
  context.value.globalStatus = 0;
  deviceList.forEach(device => {
    device.number = device.state.length;
    device.state = new Array(device.number).fill(0);
    device.IORecord.clear();
  })
  map.clear();
  pro_record.value.clear();
  context.value.clear();
}
const next = () => {
  changeState();
}

const selectedBlockProcessIndex = ref([]);

const schedulingAlgorithms = [
  {
    name: "先来先服务",
    sa: FCFS,
  },
  {
    name: "时间片轮转",
    sa: RR,
  },
  {
    name: "优先级",
    sa: PSA,

  }, {
    name: "高响应比优先",
    sa: HRRN,
  },
]
const changeSchedulingAlgorithm = (val) => {
  context.value.schedulingAlgorithm = schedulingAlgorithms[selectedSchedulingAlgorithIndex.value].sa;
}
const addProcessForm = ref({
  ID: context.value.processNum,
  name: "进程-" + context.value.processNum,
  serviceTime: 0,
  arriveTime: 0,
  priority: 0,
})
const formRef = ref(null);
const addProcess = (formEl) => {
  formEl.validate((valid) => {
    if (valid) {
      let p = new PCB(addProcessForm.value.name, addProcessForm.value.ID, addProcessForm.value.serviceTime, addProcessForm.value.priority, addProcessForm.value.arriveTime + context.value.passTime);
      let record = new Record(p);
      context.value.processNum += 1;
      if (p.arriveTime == 0) {
        context.value.readyQueues[0].push(p);
        const re = {
          "change" : "创建 => 就绪",
          "content": "进程已到达就绪队列",
        }
        record.timeMap.set(context.value.passTime, re);
      } else {
        context.value.newQueue.push(p);
        const re = {
          "change" : "无状态转换",
          "content": "进程未到达系统中",
        }
        record.timeMap.set(context.value.passTime, re);
      }
      context.value.recordQueue.push(record);
      pro_record.value.set(p, record);
      addProcessForm.value.serviceTime = 0;
      addProcessForm.value.arriveTime = 0;
      addProcessForm.value.priority = 0;
      resetAddProcessForm();
    } else {
      return false;
    }
  })
}
const resetAddProcessForm = () => {
  addProcessForm.value.serviceTime = 0;
  addProcessForm.value.arriveTime = 0;
  addProcessForm.value.priority = 0;

}
//深层监听
watch(() => context.value.processNum, (newNum, oldNum) => {
  addProcessForm.value.ID = newNum;
  addProcessForm.value.name = '进程-' + newNum;
});
const dialogFormVisible = ref(false);

let deviceContent = ref("未被占用");
const getPro = (idx, name) => {
  for (let i = 0; i < context.value.recordQueue.length; i++) {
    let rec = context.value.recordQueue[i];
    if (rec.process !== undefined && rec.process !== null) {
      if (rec.process.IoReq !== undefined && rec.process.IoReq !== null && idx >= 0) {
        if (rec.process.IoReq.device_id === idx && rec.process.IoReq.device === name) {
          deviceContent.value = `被进程 ${rec.process.name} 占用`;
          // console.log(deviceContent.value)
          return;
        }
      }
    }
  }
  deviceContent.value = "未被占用";
}

</script>

<template>
  <div id="page">
    <div style="font-size: 24px;font-weight: 500;display: flex;justify-content: center;margin-bottom: 5px;">
      <span v-if="context.globalStatus === 0">进程状态转换模拟</span>
      <span v-else>
        <span v-if="context.globalStatus !== 0">{{ schedulingAlgorithms[selectedSchedulingAlgorithIndex].name
        }}</span>
        <span v-show="context.globalStatus !== 0" style="margin-left: 10px;">{{ context.passTime }} 秒</span>
      </span>
    </div>
    <!-- 头部 -->
    <div style="font-size: 24px;font-weight: 500;position: absolute; right: 50px;top: 5px">
      <span style="margin-left:10px;">
        <el-button round type="primary" style="font-size: 20px;"
          @click="selectedSchedulingAlgorithIndex = 0; dialogFormVisible = true" v-if="context.globalStatus === 0">
          开始
        </el-button>
        <el-dialog v-model="dialogFormVisible" title="系统初始化" center width="500" style="border-radius: 10px">
          <el-form label-width="170px">
            <el-form-item label="选择调度算法">
              <el-select v-model="selectedSchedulingAlgorithIndex" placeholder="选择调度算法">
                <el-option v-for="(item, index) in schedulingAlgorithms" :key="item.name" :label="item.name"
                  :value="index" />
              </el-select>
            </el-form-item>
            <el-form-item label="选择初始线程">
              <el-select v-model="selectedInitProcessDataIndex" placeholder="选择初始线程">
                <el-option v-for="(item, index) in initData" :key="item.name" :label="item.name" :value="index" />
              </el-select>
            </el-form-item>
<!--            时间片轮转-->
            <el-form-item v-show="selectedSchedulingAlgorithIndex === 1" label="时间片大小">
              <el-input-number v-model="RR_timeSlice" :min="1">
              </el-input-number>
            </el-form-item>
          </el-form>
          <template #footer>
            <span class="dialog-footer">
              <el-button type="primary" @click="dialogFormVisible = false; context.globalStatus = 1; start()">
                自动模拟
              </el-button>
              <el-button type="warning" @click="dialogFormVisible = false; context.globalStatus = 2; start()">
                手动模拟
              </el-button>
            </span>
          </template>
        </el-dialog>
      </span>
    </div>

    <div v-if="context.globalStatus !== 0" style="position: absolute; right: 20px;top: 2px">
      <el-button style="margin:10px" type="success" @click="next" v-if="context.globalStatus === 2">下一秒
      </el-button>
      <el-button style="margin:10px" type="success" @click="continuee" v-if="context.globalStatus === 2">继续
      </el-button>
      <el-button style="margin:10px" type="info" @click="pause" v-if="context.globalStatus === 1">暂停
      </el-button>
      <el-button style="margin:10px" type="danger" @click="end" v-if="context.globalStatus !== 0">结束
      </el-button>
      <el-button  type="primary" @click="dialogFormVisibl2 = true">
        IO请求
      </el-button>
      <el-dialog v-model="dialogFormVisibl2" title="选择IO请求的设备" width="500px">
        <el-form :model="form">
          <el-form-item label="设备名称" :label-width="formLabelWidth">
            <el-select v-model="form.name" placeholder="选择其中一个设备" >
              <el-option label="键盘" value="keyboard" />
              <el-option label="打印机" value="printer" />
            </el-select>
          </el-form-item>
          <el-form-item label="设备占用时间"  :label-width="formLabelWidth">
            <el-input-number v-model="form.time" :min="1" />
          </el-form-item>
        </el-form>
        <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogFormVisibl2 = false">取消</el-button>
        <el-button type="primary" @click="dialogFormVisibl2 = false; doBlock1();" >
         确定
        </el-button>
      </span>
        </template>
      </el-dialog>
      <el-button  type="primary" style="margin-left: 16px" @click="drawerRecord = true">
        记录
      </el-button>
      <el-drawer v-model="drawerRecord" title="进程状态转换记录" direction="btt" size="600">
        <el-tabs  style="width: 1200px; margin: 0 auto;">
          <el-tab-pane label="进程">
            <el-tabs tab-position="left" style="height: 430px;" type="border-card">
              <el-tab-pane v-for="record in context.recordQueue" :label=record.process.name>
                <el-table
                  :data="Array.from(record.timeMap, ([key, value]) => ({ key, value }))"
                  style="width: 100%" height="400" border>
                  <el-table-column
                    align="center"
                    prop="key"
                    label="完成任务时刻"
                    width="150"
                  >
                    <template v-slot="{ row }">
                      <div :class="getClass(row.value['change'])" style="line-height: 65px" >{{ row.key }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column
                    align="center"
                    prop="value[change]"
                    label="进程状态转换"
                    width="300"
                  >
                    <template v-slot="{ row }">
                      <div :class="getClass(row.value['change'])" style="line-height: 65px; color:white;" >{{ row.value["change"] }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column
                    prop="value[content]"
                    label="任务主要内容">
                    <template v-slot="{ row }">
                      <div :class="getClass(row.value['change'])"  style="position: relative;" >
                        <div style="position: absolute; left: 50%; top: 50%; transform: translate(-50%,-50%);">{{ row.value["content"] }}</div>
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
            </el-tabs>
          </el-tab-pane>
          <el-tab-pane label="设备" >
            <el-tabs tab-position="left" style="height: 430px" type="border-card">
              <el-tab-pane v-for="device in deviceList" :label="device.name">
                <el-table
                  :data="Array.from(device.IORecord, ([key, value]) => ({ key, value }))"
                  style="width: 100%" height="400" border>
                  <el-table-column align="center" prop="key" label="完成任务时刻" width="150"></el-table-column>
                  <el-table-column align="center" prop="value" label="任务主要内容"></el-table-column>
                </el-table>
              </el-tab-pane>
            </el-tabs>
          </el-tab-pane>
        </el-tabs>
      </el-drawer>
    </div>
    <hr color="#bebdbd">
    <table class="CC">
      <tr>
        <td>    <!-- 队列部分 -->
          <div style="width:100%;margin-bottom:5px;">
            <table class="queuePart">
              <tr>
                <td style="padding: 0 10px">新建队列</td>
                <td>
                  <div class="queue">
                    <el-popover v-for="(item) in context.newQueue" :hide-after="50" placement="top-start" :width="200"
                                trigger="hover">
                      <template #reference>
                        <el-tag :class="getClass1(item)" style="margin:1px 10px;font-size: 16px" type="info" size="large" effect="dark" disable-transitions>
                          {{item.name}} </el-tag>
                      </template>
                      <!--显示在下方的表格-->
                      <table>
                        <tr class="hover-red">
                          <td>ID</td>
                          <td>{{ item.ID }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>名称</td>
                          <td>{{ item.name }}</td>
                        </tr>
                        <tr v-if="selectedSchedulingAlgorithIndex === 2" class="hover-red">
                          <td>优先级</td>
                          <td>{{ item.priority }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>到达时间</td>
                          <td>{{ item.arriveTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>需要时间</td>
                          <td>{{ item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>已运行时间</td>
                          <td>{{ item.useCPUTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>等待时间</td>
                          <td>{{ item.waitTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>状态</td>
                          <td>{{ item.status }}</td>
                        </tr>
                      </table>
                    </el-popover>
                  </div>
                </td>
              </tr>
              <tr>
                <td >就绪队列</td>
                <td>
                  <div class="queue">
                    <el-popover v-for="(item) in context.readyQueues[0]" :hide-after="50" placement="top-start" :width="200" trigger="hover">
                      <template #reference>
                        <el-tag  :class="getClass1(item)" style="margin: 10px;font-size: 16px;" type="info" size="large" effect="dark" disable-transitions>
                          {{item.name}}</el-tag>
                      </template>
                      <table>
                        <tr class="hover-red">
                          <td>ID</td>
                          <td>{{ item.ID }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>名称</td>
                          <td>{{ item.name }}</td>
                        </tr>
                        <tr v-if="selectedSchedulingAlgorithIndex === 2" class="hover-red">
                          <td>优先级</td>
                          <td>{{ item.priority }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>到达时间</td>
                          <td>{{ item.arriveTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>需要时间</td>
                          <td>{{ item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>已运行时间</td>
                          <td>{{ item.useCPUTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>等待时间</td>
                          <td>{{ item.waitTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>状态</td>
                          <td>{{ item.status }}</td>
                        </tr>
                        <tr v-if="item.IoReq !== null" class="hover-red">
                          <td>资源</td>
                          <td>{{ item.IoReq.device }} -- {{item.IoReq.device_id}}</td>
                        </tr>
                      </table>
                    </el-popover>
                  </div>
                </td>
              </tr>
              <tr>
                <td >运行队列 <div
                  v-if="selectedSchedulingAlgorithIndex === 1">时间片: {{ RR_timeSlice
                  }}秒</div></td>
                <td>
                  <div class="queue">
                    <el-popover v-for="(item) in context.runQueue" :hide-after="50" placement="top-start" :width="200"
                                trigger="hover">
                      <template #reference>
                        <el-badge :value="item.useCPUTime"  type="primary">
                          <el-tag :class="getClass1(item)" style="margin: 1px 10px;font-size: 16px;" type="info" size="large" effect="dark" disable-transitions>
                            {{item.name}} </el-tag>
                        </el-badge>
                      </template>
                      <table>
                        <tr class="hover-red">
                          <td>ID</td>
                          <td>{{ item.ID }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>名称</td>
                          <td>{{ item.name }}</td>
                        </tr>
                        <tr v-if="selectedSchedulingAlgorithIndex === 2" class="hover-red">
                          <td>优先级</td>
                          <td>{{ item.priority }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>到达时间</td>
                          <td>{{ item.arriveTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>需要时间</td>
                          <td>{{ item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>已运行时间</td>
                          <td>{{ item.useCPUTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>等待时间</td>
                          <td>{{ item.waitTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>状态</td>
                          <td>{{ item.status }}</td>
                        </tr>
                        <tr v-if="item.IoReq !== null" class="hover-red">
                          <td>资源</td>
                          <td>{{ item.IoReq.device }} -- {{item.IoReq.device_id}}</td>
                        </tr>
                      </table>
                    </el-popover>
                  </div>
                </td>
              </tr>
              <tr>
                <td >阻塞队列</td>
                <td>
                  <div class="queue">
                    <el-popover v-for="(item) in context.blockQueue" :hide-after="50" placement="top-start" :width="400"
                                trigger="hover">
                      <template #reference>
                        <!--                  如果不是等待设备的阻塞-->
                        <template v-if="!item.isIO">
                          <el-tag
                            type="info" size="large" effect="dark"
                            v-if="item.IoReq.device ==='printer'"
                            style="color: white;background-color: lightblue; margin: 10px;font-size: 16px;">
                            {{ item.name }}</el-tag>
                          <el-tag
                            type="info" size="large" effect="dark"
                            v-if="item.IoReq.device === 'keyboard'"
                            style="color: white;background-color: lightgreen; margin: 10px;font-size: 16px;">
                            {{ item.name }}</el-tag>
                        </template>
                        <!--                  等待设备的阻塞-->
                        <template v-else>
                          <el-tag
                            type="info" size="large" effect="dark"
                            v-if="item.IoReq.device ==='printer' "
                            style="color: white;background-color: #4e4eab;border: 3px solid #c94444; margin: 10px;font-size: 15px;">
                            {{ item.name }}</el-tag>
                          <el-tag
                            type="info" size="large" effect="dark"
                            v-if="item.IoReq.device === 'keyboard'"
                            style="color: white;background-color: #3c9d3c;border: 3px solid #da5e5e; margin: 10px;font-size: 16px;">
                            {{ item.name }}</el-tag>
                        </template>
                      </template>
                      <table>
                        <tr class="hover-red">
                          <td>ID</td>
                          <td>{{ item.ID }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>名称</td>
                          <td>{{ item.name }}</td>
                        </tr>
                        <tr v-if="selectedSchedulingAlgorithIndex === 2" class="hover-red">
                          <td>优先级</td>
                          <td>{{ item.priority }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>到达时间</td>
                          <td>{{ item.arriveTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>需要时间</td>
                          <td>{{ item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>已运行时间</td>
                          <td>{{ item.useCPUTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>等待时间</td>
                          <td>{{ item.waitTime }}</td>
                        </tr>
                        <tr v-if="!item.isIO" class="hover-red">
                          <td>I/O结束时间</td>
                          <td>{{ item.wakeTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>状态</td>
                          <td>{{ item.status }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>阻塞原因</td>
                          <td v-if="item.isIO">申请设备{{item.IoReq.device}}失败，等待空闲设备</td>
                          <td v-else>申请设备{{item.IoReq.device}}成功，正在等待I/O请求结束</td>
                        </tr>
                        <tr v-if="!item.isIO" class="hover-red">
                          <td>资源</td>
                          <td>{{item.IoReq.device}} -- {{item.IoReq.device_id}}</td>
                        </tr>
                      </table>
                    </el-popover>
                  </div>
                </td>
              </tr>
              <tr>
                <td >结束进程</td>
                <td>
                  <div class="queue">
                    <el-popover v-for="(item) in context.finishQueue" :hide-after="50" placement="top-start" :width="200"
                                trigger="hover">
                      <template #reference>
                        <el-tag :class="getClass1(item)" style="margin: 10px;font-size: 16px;" type="info" size="large" effect="dark" disable-transitions> {{item.name}} </el-tag>
                      </template>
                      <table>
                        <tr class="hover-red">
                          <td>ID</td>
                          <td>{{ item.ID }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>名称</td>
                          <td>{{ item.name }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>优先级</td>
                          <td>{{ item.priority }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>到达时间</td>
                          <td>{{ item.arriveTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>需要时间</td>
                          <td>{{ item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>已运行时间</td>
                          <td>{{ item.useCPUTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>等待时间</td>
                          <td>{{ item.waitTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>周转时间</td>
                          <td>{{ item.waitTime + item.serviceTime }}</td>
                        </tr>
                        <tr class="hover-red">
                          <td>状态</td>
                          <td>{{ item.status }}</td>
                        </tr>
                      </table>
                    </el-popover>
                  </div>
                </td>
              </tr>
            </table>
          </div>
        </td>
        <td>
          <div
            style="border:1px solid black;border-radius: 5px;margin-right:5px;height:343px;overflow-x: auto" >
            <el-tabs type="border-card" style="display: inline-block;" >
              <el-tab-pane label="新建队列">
                <el-table :data="context.newQueue"  height="260" border style="width: 100%">
                  <el-table-column align="center" prop="ID" label="ID" width="60">
                    <template v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.ID }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="name" label="名称"  width="90">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.name }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 2" align="center" prop="priority" label="优先级"  width="90">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" style="font-weight: bolder" class="hover-red">{{ row.priority }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="arriveTime" label="到达时间"  width="100">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.arriveTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="serviceTime" label="需要时间" width="100" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="useCPUTime" label="已运行时间" width="110" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.useCPUTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="waitTime" label="等待时间" width="100" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.waitTime }}</div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
              <el-tab-pane label="就绪队列">
                <el-table :data="context.readyQueues[0]"  height="260" border style="width: 100%;">
                  <el-table-column align="center" prop="ID" label="ID" width="60">
                    <template v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.ID }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="name" label="名称"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.name }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 2" align="center" prop="priority" label="优先级"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" style="font-weight: bolder" class="hover-red">{{ row.priority }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 3" align="center" label="响应比"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red" style="font-weight: bolder">
                        {{  (1 + (row.waitTime / row.serviceTime)).toFixed(3)  }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="arriveTime" label="到达时间"  width="85">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.arriveTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="serviceTime" label="需要时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="useCPUTime" label="已运行时间" width="95" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.useCPUTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="waitTime" label="等待时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.waitTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" label="资源状况" width="105" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" v-if="row.IoReq !== null && row.IoReq !== undefined" class="hover-red">
                        {{ row.IoReq.device_id !== undefined ? row.IoReq.device + "--" + row.IoReq.device_id : "未拥有资源" }}
                      </div>
                      <div  @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" v-else>
                        无
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
              <el-tab-pane label="运行队列">
                <el-table :data="context.runQueue"  height="260" border style="width: 100%">
                  <el-table-column align="center" prop="ID" label="ID" width="60">
                    <template v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.ID }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="name" label="名称"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.name }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 2" align="center" prop="priority" label="优先级"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" style="font-weight: bolder" class="hover-red">{{ row.priority }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 3" align="center" label="响应比"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red" style="font-weight: bolder">
                        {{  (1 + (row.waitTime / row.serviceTime)).toFixed(3) }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="arriveTime" label="到达时间"  width="85">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.arriveTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="serviceTime" label="需要时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="useCPUTime" label="已运行时间" width="95" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.useCPUTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="waitTime" label="等待时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.waitTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" label="资源状况" width="105" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" v-if="row.IoReq !== null && row.IoReq !== undefined" class="hover-red">
                        {{ row.IoReq.device_id !== undefined ? row.IoReq.device + "--" + row.IoReq.device_id : "未拥有资源" }}
                      </div>
                      <div  @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" v-else>
                        无
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
              <el-tab-pane label="阻塞队列">
                <el-table :data="context.blockQueue"  height="260" border style="width: 100%">
                  <el-table-column align="center" prop="ID" label="ID" width="60">
                    <template v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.ID }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="name" label="名称"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.name }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 2" align="center" prop="priority" label="优先级"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" style="font-weight: bolder" class="hover-red">{{ row.priority }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column v-if="selectedSchedulingAlgorithIndex === 3" align="center" label="响应比"  width="80">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red" style="font-weight: bolder">
                        {{  (1 + (row.waitTime / row.serviceTime)).toFixed(3) }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="arriveTime" label="到达时间"  width="85">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.arriveTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="serviceTime" label="需要时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="useCPUTime" label="已运行时间" width="95" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.useCPUTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="waitTime" label="等待时间" width="85" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.waitTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" label="资源状况" width="105" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" v-if="row.IoReq !== null && row.IoReq !== undefined" class="hover-red">
                        {{ row.IoReq.device_id !== undefined ? "正在申请 " + row.IoReq.device + "--" + row.IoReq.device_id : "等待申请中" }}
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
              <el-tab-pane label="终止队列">
                <el-table :data="context.finishQueue"  height="260" border style="width: 100%">
                  <el-table-column align="center" prop="ID" label="ID" width="60">
                    <template v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.ID }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="name" label="名称"  width="90">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.name }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="arriveTime" label="到达时间"  width="90">
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.arriveTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="serviceTime" label="需要时间" width="90" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="useCPUTime" label="已运行时间" width="100" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.useCPUTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" prop="waitTime" label="等待时间" width="90" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{ row.waitTime }}</div>
                    </template>
                  </el-table-column>
                  <el-table-column align="center" label="周转时间" width="90" >
                    <template  v-slot="{row}" >
                      <div @mouseenter="row.highlight = true" @mouseleave="row.highlight = false" class="hover-red">{{row.waitTime + row.serviceTime }}</div>
                    </template>
                  </el-table-column>
                </el-table>
              </el-tab-pane>
            </el-tabs>
          </div>
        </td>
      </tr>
      <tr>
        <td>
          <div style="">
            <!--    显示设备管理-->
            <el-table :data="deviceList" stripe border style="color: black">
              <el-table-column align="center" prop="name" label="设备名称" width="100" />
              <el-table-column align="center" prop="number" label="设备剩余数量" width="120" />
              <el-table-column align="center" prop="state" label="设备状态" >
                <template v-slot="{ row }">
                  <span v-for="(st,idx) in row.state" style="margin: 3px;">
                      <el-tooltip  placement="bottom" effect="light">
                        <template v-slot:content>
                            <span>{{ deviceContent }}</span>
                        </template>
                        <el-button @mouseenter="() => getPro(idx, row.name)" v-if="st === 0"> 空闲 </el-button>
                        <el-button @mouseenter="() => getPro(idx, row.name)" v-else style="background-color: #e03e3e;color: white;">  占用 </el-button>
                      </el-tooltip>
                  </span>
                </template>
              </el-table-column>
            </el-table>
          </div>
        </td>
        <td>
          <div style="height:290px; text-align: center;">
            <div id="create-process-area" style="height: 100%;border-radius: 5px;">
              <div style="font-size: 25px;">创建进程</div>
              <el-form ref="formRef" size="small" label-position="right" label-width="100px" :model="addProcessForm"
                       style="padding:10px;">
                <el-form-item label="进程ID：">
                  <el-input style="width:120px" type="text" v-model.number="addProcessForm.ID" disabled />
                </el-form-item>
                <el-form-item label="进程名：">
                  <el-input style="width:120px" type="text" v-model="addProcessForm.name" disabled />
                </el-form-item>
                <el-form-item label="到达时间：">
<!--                  <el-input-number v-model="addProcessForm.arriveTime" :min="1"></el-input-number>-->
                  <el-slider v-model="addProcessForm.arriveTime" :min="1" size="small" style="width: 140px"></el-slider>
                  <span style="margin-left:5px">秒后</span>
                </el-form-item>
                <el-form-item label="需要时间：">
<!--                  <el-input-number v-model="addProcessForm.serviceTime" :min="1"></el-input-number>-->
                  <el-slider v-model="addProcessForm.serviceTime" :min="1" size="small" style="width: 140px"></el-slider>
                </el-form-item>
                <el-form-item label="优先级：">
<!--                  <el-input-number v-model="addProcessForm.priority" :min="1"></el-input-number>-->
                  <el-slider v-model="addProcessForm.priority" :min="1" size="small" style="width: 140px" />
                </el-form-item>
                <el-form-item>
                  <el-button :disabled="context.globalStatus === 0" type="primary" @click="addProcess(formRef)">添加
                  </el-button>
                  <el-button :disabled="context.globalStatus === 0" @click="resetAddProcessForm()">重置</el-button>
                </el-form-item>
              </el-form>
            </div>
          </div>
        </td>
      </tr>
    </table>
  </div>
</template>


<style lang="less">
/* @import './assets/css/styles.less'; */

.withHighlight {
  background-color: #0a37ec;
  color: white;
}

.withPrinter {
  background-color: lightblue;
}

.withKeyboard {
  background-color: lightgreen;
}

.no {
  border-radius: 5px;
  background-color: #c26dec;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  font-weight: bold;
}

.rr {
  border-radius: 5px;
  background-color: lightblue;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  //color: #1e4198;
  font-weight: bold;
}
.rr1 {
  border-radius: 5px;
  background-color: lightgoldenrodyellow;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  //color: #1e4198;
  font-weight: bold;
}
.rb {
  border-radius: 5px;
  background-color: lightgreen;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  font-weight: bold;
}
.br {
  border-radius: 5px;
  background-color: lightcoral;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  font-weight: bold;
}
.cr {
  border-radius: 5px;
  background-color: lightgray;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  font-weight: bold;
}
.re {
  border-radius: 5px;
  background-color: lightseagreen;
  padding: 5px;
  box-sizing: border-box;
  height: 75px;
  font-weight: bold;
}

body {
  font-family: Roboto, -apple-system, BlinkMacSystemFont, 'Helvetica Neue', 'Segoe UI', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Open Sans', sans-serif;
}

.CC{
  td {
    margin: 3px 10px;
  }
  tr,td {
    padding: 3px;
    //border: 1px solid red;
  }
  //border: 1px solid red;
}

#page {
  text-align: center;
  //max-width: 1020px;
  //max-height: 670px;
}

.top-btn {
  vertical-align: middle;
  width: 100px;
  height: 30px;
  font-size: 22px;
  line-height: 30px;
}

.queuePart {
  //margin: 0 auto;
  border: 1px solid black;
  border-collapse: collapse;

  th,
  td {
    border-right: none; /* 去掉竖线 */
    border-bottom: 1px solid #9b9696; /* 添加横线作为分隔线，可根据需要调整样式 */
  }

  th:last-child,
  td:last-child {
    border-right: none; /* 去掉最后一列的竖线 */
  }
}



.queue {
  //border: 2px solid black;
  //border-bottom: 1px solid black;
  height: 60px;
  width: 600px;
  display: inline-flex;
  align-items: center;
  background-color: #f5f7fa;
  overflow-x: scroll;
  border-radius: 5px;
}


.queue::-webkit-scrollbar {
  width: 10px;
  height: 7px;
}

.queue::-webkit-scrollbar-thumb {
  border-radius: 10px;
  -webkit-box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.2);
  background: #cccccc;
}

.queue::-webkit-scrollbar-track {
  -webkit-box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.2);
  border-radius: 10px;
  background: #ffffff;
}

#create-process-area {
  width: 330px;
  border: 1px solid black;
}

#other-option-area {
  width: 380px;
  border: 1px solid black;
  padding-bottom: 3px
}

.option-btn {
  font-size: 17px;
}

.hover-red:hover {
  color: blue;
}
</style>
