<script setup lang="ts">
import { InputProps, MessagePlugin, TableProps } from 'tdesign-vue-next'
import { Contest, getContestNormalConfig, getRecordList, getStudentList, getTeamList } from '../../libs/core/contest'
import { RecordData, StudentData, TeamData } from '../../libs/types/basic-type'
import { buildStudentBoard, buildStudentTableData, buildTeamBoard, exportSingle, exportTeam, tableColumns, getSchoolAvatar } from '../../libs/core/board-gplt'
import { StudentBoardData, TeamBoardData } from '../../libs/types/board-gplt'
const route = useRoute()
const interval = ref<number>()
const interval_update = ref<number>()
const boardLoading = ref(true)
const currentView = ref(0)
const openModal = ref(false)
const modalTitle = ref('')
const visibleItems = ref<number[]>([])

const contestData = ref<Contest>()
const teamList = ref([] as TeamData[])
const studentList = ref([] as StudentData[])
const recordList = ref([] as RecordData[])
const boardTeamData = ref([] as TeamBoardData[])
const boardStudentData = ref([] as StudentBoardData[])
const teamViewData = ref([] as TeamBoardData[])
const studentViewData = ref([] as StudentBoardData[])
const tableData = ref([] as TableProps['data'])
const getContestData = async () => {
  const contestId = route.query.contest_id as string
  if (!contestId) {
    MessagePlugin.error('比赛 ID 为空!')
  } else {
    // get contest data
    contestData.value = await getContestNormalConfig(contestId)
    if (contestData.value) {
      teamList.value = await getTeamList(contestId)
      studentList.value = await getStudentList(contestId)
      recordList.value = await getRecordList(contestId)
      // build board data
      boardTeamData.value = buildTeamBoard(contestData.value, studentList.value, teamList.value, recordList.value)
      boardStudentData.value = buildStudentBoard(contestData.value, studentList.value, recordList.value)
      teamViewData.value = boardTeamData.value
      studentViewData.value = boardStudentData.value
      // update contest status
      useTitle(`${contestData.value!.title} | ZZULI GPLT Board`)
      contestData.value.updateContestStatus()
      interval.value = setInterval(() => {
        contestData.value!.updateContestStatus()
      }, 1000)
      if (contestData.value.getContestStatus() === ContestStatus.RUNNING) {
        interval_update.value = setInterval(async () => {
          if (contestData.value) {
            teamList.value = await getTeamList(contestId)
            studentList.value = await getStudentList(contestId)
            recordList.value = await getRecordList(contestId)
            // build board data
            boardTeamData.value = buildTeamBoard(contestData.value, studentList.value, teamList.value, recordList.value)
            boardStudentData.value = buildStudentBoard(contestData.value, studentList.value, recordList.value)
          }
        }, 30000)
      }
    }
  }
  boardLoading.value = false
}
const onChangeMode = async (mode: number) => {
  teamViewData.value = boardTeamData.value
  studentViewData.value = boardStudentData.value
  currentView.value = mode
  await updateVisibleStatus()
}
const onTeamSearch: InputProps['onChange'] = async (value) => {
  if (value === '') {
    teamViewData.value = boardTeamData.value
  } else {
    teamViewData.value = boardTeamData.value.filter((item) => item.team_name.includes(value.toString()))
  }
  await updateVisibleStatus()
}
const onStudentSearch: InputProps['onChange'] = async (value) => {
  if (value === '') {
    studentViewData.value = boardStudentData.value
  } else {
    studentViewData.value = boardStudentData.value.filter((item) => item.name.includes(value.toString()))
  }
  await updateVisibleStatus()
}
const onTeamRowClick = (team_id: number) => {
  const team = teamList.value.find((item) => item.id === team_id)
  modalTitle.value = `${team?.name} ————— ${team?.college} ————— ${team?.class}`
  const students = boardStudentData.value.filter((item) => item.team_id === team_id)
  tableData.value = buildStudentTableData(students)
  openModal.value = true
}
const onStudentRowClick = (member_id: string) => {
  const student = boardStudentData.value.filter((item) => item.member_id === member_id)
  modalTitle.value = `${student[0].name} ————— ${student[0].college} ————— ${student[0].class}`
  tableData.value = buildStudentTableData(student)
  openModal.value = true
}

/* ---------------- common functions ---------------- */
const updateVisibleStatus = async () => {
  visibleItems.value = []
  await nextTick()
  await checkVisibility()
}
const isElementVisible = (index: number) => {
  const target = document.getElementById(`node_${index}`)
  if (target) {
    const rect = target.getBoundingClientRect()
    const windowHeight = window.innerHeight || document.documentElement.clientHeight
    const isVisible = rect.top < windowHeight && rect.bottom >= 0
    return isVisible
  }
  return false
}
const checkVisibility = async () => {
  if (currentView.value === 2) return
  let target = currentView.value === 0 ? teamViewData.value : studentViewData.value
  target.forEach((_, index) => {
    if (isElementVisible(index)) {
      if (!visibleItems.value.includes(index)) {
        visibleItems.value.push(index)
      }
    } else {
      const itemIndex = visibleItems.value.indexOf(index)
      if (itemIndex !== -1) {
        visibleItems.value.splice(itemIndex, 1)
      }
    }
  })
}
onMounted(async () => {
  await getContestData()
  await updateVisibleStatus()
  window.addEventListener('scroll', checkVisibility)
})
onUnmounted(() => {
  clearInterval(interval.value)
  clearInterval(interval_update.value)
  window.removeEventListener('scroll', checkVisibility)
})
</script>

<template>
  <Loading v-if="boardLoading" />
  <t-empty v-else-if="!contestData" size="large" />
  <div v-else flex flex-col gap-3 items-center>
    <!-- 比赛标题 -->
    <span text-3xl font-700>{{ contestData?.title }}</span>
    <!-- 比赛状态条 -->
    <div flex flex-col flex-1 gap-0.4rem w-1270px>
      <div flex flex-row justify-center items-center gap-3>
        <div class="circle" :data-status="contestData?.contestStatus"></div>
        <strong text-18px font-800>{{ contestData?.statusStr }}</strong>
      </div>
      <div class="progress">
        <div :data-status="contestData?.contestStatus" class="progress-warrper" :style="{ width: `${contestData?.progressRatio}%` }"></div>
      </div>
    </div>
    <!-- 工具选项栏 -->
    <div grid grid-cols-2 w-1270px mt-1>
      <div flex gap-4 align-start>
        <t-button :variant="currentView === 0 ? 'outline' : 'dashed'" ghost @click="onChangeMode(0)">团队排名</t-button>
        <t-button :variant="currentView === 1 ? 'outline' : 'dashed'" ghost @click="onChangeMode(1)">个人排名</t-button>
      </div>
      <div flex gap-4 justify-end>
        <t-button :variant="currentView === 2 ? 'outline' : 'dashed'" ghost @click="onChangeMode(2)">数据导出</t-button>
      </div>
    </div>
    <!-- 搜索框 -->
    <div w-1270px backdrop-blur rounded-md bg-gray-700 bg-op-30>
      <t-input v-if="currentView === 0" size="large" placeholder="搜索队伍名称" @change="onTeamSearch"></t-input>
      <t-input v-if="currentView === 1" size="large" placeholder="搜索学生名称" @change="onStudentSearch"></t-input>
    </div>
    <div box-border w-1270px backdrop-blur border-1 border-blue-600 border-solid rounded-md bg-gray-700 bg-op-30>
      <div v-if="currentView === 0" v-for="(item, index) in teamViewData" :key="item.team_id" :id="`node_${index}`" box-border flex flex-col gap-2 w-1270px h-115px py-20px px-10px odd:bg-gray-600 odd:bg-op-30>
        <div v-if="visibleItems.includes(index)" @click="onTeamRowClick(item.team_id)">
          <div pl-210px>
            <span>{{ item.team_name }} ——— {{ item.college }} ——— {{ item.class }}</span>
          </div>
          <div box-border flex items-center w-1270px>
            <div inline-flex justify-center items-center w-160px>
              <span text-yellow font-800 italic text-3xl>{{ item.rank }}</span>
            </div>
            <div class="p-head">
              <img :src="getSchoolAvatar(item.school)" alt="school" size-40px my-5px mx-5px />
            </div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part1.score} / 1000`">
                <img src="/process-level1.png" alt="l1" class="p-level" :style="`width:${item.part1.ratio}%`" />
                <img src="/process-level1-node.png" alt="l1-node" class="p-level-node" />
              </t-tooltip>
            </div>
            <div class="p-node"></div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part2.score} / 1000`">
                <img v-if="item.part1.status" src="/process-level2.png" alt="l2" class="p-level" :style="`width:${item.part2.ratio}%`" />
                <img v-else src="/process-level0.png" alt="l2" class="p-level" :style="`width:${item.part2.ratio}%`" />
              </t-tooltip>
              <img v-if="item.part1.status" src="/process-level2-node.png" alt="l2-node" class="p-level-node" />
            </div>
            <div class="p-node"></div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part3.score} / 900`">
                <img v-if="item.part2.status" src="/process-level3.png" alt="l3" class="p-level" :style="`width:${item.part3.ratio}%`" />
                <img v-else src="/process-level0.png" alt="l3" class="p-level" :style="`width:${item.part3.ratio}%`" />
              </t-tooltip>
              <img v-if="item.part2.status" src="/process-level3-node.png" alt="l3-node" class="p-level-node" />
            </div>
            <div class="p-end"></div>
            <div inline-flex justify-center items-center w-160px>
              <span text-yellow font-800 italic text-3xl>{{ item.sum }}</span>
            </div>
          </div>
        </div>
      </div>
      <div v-else-if="currentView === 1" v-for="(item, index) in studentViewData" :key="item.member_id" box-border flex flex-col gap-2 w-1270px py-20px px-10px odd:bg-gray-600 odd:bg-op-30 :id="`node_${index}`" h-115px>
        <div v-if="visibleItems.includes(index)" @click="onStudentRowClick(item.member_id)">
          <div pl-210px>
            <span>{{ item.name }} ——— {{ item.college }} ——— {{ item.class }}</span>
          </div>
          <div box-border flex items-center w-1270px>
            <div inline-flex justify-center items-center w-160px>
              <span text-yellow font-800 italic text-3xl>{{ item.rank }}</span>
            </div>
            <div class="p-head">
              <img :src="getSchoolAvatar(item.school)" alt="school" size-40px my-5px mx-5px />
            </div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part1.score} / 100`">
                <img src="/process-level1.png" alt="l1" class="p-level" :style="`width:${item.part1.ratio}%`" />
                <img src="/process-level1-node.png" alt="l1-node" class="p-level-node" />
              </t-tooltip>
            </div>
            <div class="p-node"></div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part2.score} / 100`">
                <img src="/process-level2.png" alt="l2" class="p-level" :style="`width:${item.part2.ratio}%`" />
              </t-tooltip>
              <img src="/process-level2-node.png" alt="l2-node" class="p-level-node" />
            </div>
            <div class="p-node"></div>
            <div class="p-pipe">
              <t-tooltip placement="bottom" :content="`${item.part3.score} / 90`">
                <img src="/process-level3.png" alt="l3" class="p-level" :style="`width:${item.part3.ratio}%`" />
              </t-tooltip>
              <img src="/process-level3-node.png" alt="l3-node" class="p-level-node" />
            </div>
            <div class="p-end"></div>
            <div inline-flex justify-center items-center w-160px>
              <span text-yellow font-800 italic text-3xl>{{ item.sum }}</span>
            </div>
          </div>
        </div>
      </div>
      <div v-else flex gap-20 justify-center m-20>
        <t-button theme="primary" size="large" @click="exportTeam(boardTeamData, boardStudentData, contestData!.title)">
          <template #icon>
            <CloudDownloadIcon />
          </template>
          导出团队排名
        </t-button>
        <t-button theme="primary" size="large" @click="exportSingle(boardStudentData, contestData!.title)">
          <template #icon>
            <CloudDownloadIcon />
          </template>
          导出个人排名
        </t-button>
      </div>
    </div>
  </div>
  <t-dialog v-model:visible="openModal" closeOnOverlayClick :header="false" :footer="false" :closeBtn="false" width="90%" placement="center">
    <div flex flex-col gap-3 items-center>
      <h2>{{ modalTitle }}</h2>
      <t-table row-key="member_id" :data="tableData" :columns="tableColumns" bordered> </t-table>
    </div>
  </t-dialog>
</template>

<style scoped>
.p-node {
  position: relative;
  display: inline-block;
  vertical-align: middle;
  width: 25px;
  height: 40px;
  background-image: url('/process-node1.png');
  background-repeat: no-repeat;
}

.p-level-node {
  position: absolute;
  display: inline-block;
  vertical-align: middle;
  z-index: 10;
  margin-top: -13px;
  margin-left: -24px;
}

.p-level {
  position: relative;
  display: inline-block;
  vertical-align: middle;
  z-index: 1;
  top: 4px;
  height: 20px;
}

.p-pipe {
  position: relative;
  display: inline-block;
  vertical-align: middle;
  width: 260px;
  height: 28px;
  background-image: url('/process-pipe.png');
}

.p-end {
  position: relative;
  display: inline-block;
  vertical-align: middle;
  width: 45px;
  height: 50px;
  background-image: url('/process-end.png');
}

.p-head {
  position: relative;
  display: inline-block;
  vertical-align: middle;
  width: 45px;
  height: 50px;
  background-image: url('/process-head.png');
}

.circle {
  width: 0.5em;
  height: 0.5em;
  border-radius: 50%;
}

.circle[data-status='0'] {
  background-color: #a3a3a3;
}

.circle[data-status='1'] {
  background-color: #0369a1;
}

.circle[data-status='2'] {
  background-color: #eab308;
}

.circle[data-status='3'] {
  background-color: #22c55e;
}

.circle[data-status='4'] {
  background-color: #dc2626;
}

@keyframes progressbar {
  0% {
    background-position: 40px 0;
  }

  100% {
    background-position: 0 0;
  }
}

.progress {
  height: 1rem;
  background: #e4e4e4;
  position: relative;
}

.progress-warrper {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.15) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.15) 75%, transparent 75%, transparent);
  background-color: #006ace;
  background-size: 40px 40px;
  animation: progressbar 2s linear infinite;
  transition: all 0.5s ease;
}

.progress-warrper[data-status='0'] {
  background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.15) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.15) 75%, transparent 75%, transparent);
  background-color: #3d3d3d;
}

.progress-warrper[data-status='2'] {
  background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.15) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.15) 75%, transparent 75%, transparent);
  background-color: #f18d09;
}

.progress-warrper[data-status='3'] {
  background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.15) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.15) 75%, transparent 75%, transparent);
  background-color: #14cc2c;
}

.progress-warrper[data-status='4'] {
  background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.15) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.15) 75%, transparent 75%, transparent);
  background-color: #cf4242;
}
</style>

<route lang="yaml">
meta:
  layout: board-gplt
</route>
