<template>
  <main>
    <section
      v-if="!degree_evaluation_was_uploaded"
      class="w-full text-center h-screen flex flex-col gap-4 justify-center items-center"
    >
      <h1 class="font-title font-medium text-4xl">Upload your Student Evaluation</h1>
      <p class="font-text font-medium">Upload your student evaluation file as an MHTML file</p>
      <p class="max-w-3xl leading-8">
        To convert your degree evaluation to a .mhtml format, go to the degree evaluation page, and
        click CTRL S. Make sure to save the webpage as one file (in .mhtml format). Your degree
        evaluation file is not saved anywhere (except in the process of extracting your courses,
        after which it is permanently deleted)
      </p>

      <FileUpload
        ref="fileupload"
        mode="basic"
        name="degree_eval"
        :url="upload_url"
        @upload="onUpload"
        class="!bg-udmercy-blue !border-udmercy-blue"
      />
      <button
        class="bg-udmercy-blue cursor-pointer font-semibold text-sm px-6 py-3 rounded-md mt-1 font-text text-white"
        @click="upload"
      >
        Upload File
      </button>
      <p v-if="uploading_file">Uploading File...</p>
    </section>
    <section v-else>
      <section
        class="flex items-center flex-col justify-center h-screen"
        v-if="!course_history_verified"
      >
        <div class="text-center flex flex-col gap-5 px-15">
          <h1 class="font-title font-medium text-4xl">Verify your Course History</h1>
          <p class="text-md leading-8">
            The course extraction may not be perfect. Look through the courses to make sure they
            match, or at least look good enough
          </p>
          <section class="flex justify-center gap-20">
            <article class="w-1/3">
              <p class="font-medium text-xl underline mb-5">Requirements Not Satisfied:</p>
              <div class="max-h-[50vh] overflow-auto px-5">
                <details v-for="requirement in requirements_not_satisfied" class="mb-5">
                  <summary class="font-text cursor-pointer text-md font-medium leading-8">
                    {{ requirement.caption }}
                  </summary>
                  <div class="mt-5">
                    <p>Missing Courses: {{ requirement.courses.join(', ') }}</p>
                  </div>
                </details>
              </div>
            </article>
            <div class="h-full w-0.5 bg-gray-300 rounded-md"></div>
            <article class="max-w-1/3">
              <p class="font-medium text-xl underline mb-5">Requirements Satisfied:</p>
              <div class="max-h-[50vh] px-5 overflow-auto flex flex-wrap justify-between gap-5">
                <span
                  v-for="req in requirements_satisfied"
                  class="flex gap-4 justify-center items-center"
                >
                  <img
                    :src="CancelIcon"
                    alt="Cancel Icon"
                    class="w-3 cursor-pointer"
                    @click="() => removeSatisfiedReq(req)"
                  />
                  <p
                    class="w-22 whitespace-nowrap overflow-hidden text-ellipsis"
                    :title="req.course"
                  >
                    {{ req.course }}: {{ req.min_grade }}
                  </p>
                  <input
                    type="text"
                    v-model="req.grade"
                    class="w-10 border-b outline-0 text-center"
                  />
                </span>
              </div>
            </article>
          </section>
        </div>
        <button
          class="bg-udmercy-blue mt-5 cursor-pointer font-semibold text-sm px-6 py-3 rounded-md font-text text-white"
          @click="startAdvisorAssistant"
        >
          Looks Good!
        </button>
      </section>
      <!-- PROCESSED REQS SECTION -->
      <section class="p-5 flex gap-8" v-else>
        <div class="w-lg">
          <div class="h-[75vh] overflow-auto">
            <h1 class="font-title text-2xl mb-3 font-medium">Missing Requirements</h1>
            <div class="mb-3">
              <label for="">Hide Courses that are not Offered</label>
              <input type="checkbox" class="ml-4" v-model="hide_courses_not_offered" />
            </div>
            <p>
              Outstanding courses are automatically fetched based on either their course code or the
              necessary attributes
            </p>
            <div class="max-h-[85vh] w-96 overflow-auto">
              <div v-for="(val, key) in processed_requirements">
                <details v-if="val.length > 0 || hide_courses_not_offered === false">
                  <summary class="font-title text-xl my-2">{{ key }}</summary>
                  <div v-if="val.length > 0">
                    <div
                      v-for="course in val"
                      class="border box-border px-3 pb-5 cursor-pointer pt-4 my-2 rounded-md border-gray-300"
                      @click="() => addCourse(course)"
                      :class="{ 'selected-course': course.is_selected }"
                    >
                      <p class="font-medium overflow-ellipsis w-80">
                        {{ course.subject }} {{ course.course_number }} - {{ course.course_name }}
                      </p>
                      <p class="text-md">
                        Course Number: {{ course.course_number }} | Section {{ course.section }}
                      </p>
                      <p v-if="course.meeting_begin_time" class="text-[15px]">
                        Time:
                        {{
                          course.meeting_begin_time.slice(0, 2) +
                          ':' +
                          course.meeting_begin_time.slice(2)
                        }}
                        -
                        {{
                          course.meeting_end_time.slice(0, 2) +
                          ':' +
                          course.meeting_end_time.slice(2)
                        }}. Days: {{ formatCourseDays(course) }}.
                      </p>
                      <p v-else>This course does not have a set meeting time</p>
                      <p v-if="course.attributes && course.attributes.length > 0">
                        Attributes: {{ formatCourseAttributes(course) }}
                      </p>
                      <p v-if="course.credits">Credits: {{ course.credits }}</p>
                    </div>
                  </div>

                  <p class="text-md" v-else>Not Offered</p>
                </details>
              </div>
            </div>
          </div>
          <div>
            <h1 class="font-title text-2xl mb-3 mt-3 font-medium">Chosen Courses</h1>
            <p class="underline cursor-pointer" @click="saveAsTxtFile">
              Click to save as a .txt file
            </p>
            <ul class="mt-3 max-h-30 overflow-auto">
              <li v-for="course in chosen_courses" class="font-text mb-3">
                <p class="font-medium">{{ course.course_name }}</p>
                <p>{{ formatCourseTime(course) }}</p>
              </li>
            </ul>
          </div>
        </div>
        <div class="w-full h-screen">
          <h1 class="font-title text-2xl mb-4 font-medium">Course Calendar</h1>
          <vue-cal
            :events="events"
            selected-date="2025-05-05"
            :time-from="8 * 60"
            hide-view-selector
            :time-cell-height="60"
            hide-title-bar
            :time-to="23 * 60"
            :disable-views="['years', 'year', 'month']"
          >
          </vue-cal>
        </div>
      </section>
    </section>

    <Dialog
      v-model:visible="course_info_is_visible"
      modal
      header="Course Information"
      :style="{ width: '25rem', 'font-family': 'Raleway' }"
    >
      <p class="text-surface-500 font-semibold dark:text-surface-400 block mt-[-5px]">
        Prerequisites
      </p>
      <div class="mt-2" v-if="course_prerequisites !== 'No prerequisite information available.'">
        <p v-if="course_prerequisites.length == 0">No prerequitie information available</p>
        <div v-else v-for="req in course_prerequisites" class="leading-8">
          <p>{{ req.subject }}: {{ req.course_number }}. Minimum Grade: {{ req.min_grade }}</p>
          <p>Requirement Satisfied: {{ checkForSatisfiedReq(req.subject, req.course_number) }}</p>
          <hr class="my-3 text-gray-300" />
        </div>
      </div>
      <p class="mt-2" v-else>No prerequisites were found</p>
      <p class="text-surface-500 mt-4 font-semibold dark:text-surface-400 block">Corequisites</p>

      <p class="mt-2">
        {{
          course_corequisites === 'No co reqs information available.'
            ? 'No corequisites were found'
            : course_corequisites.join(', ')
        }}
      </p>

      <button
        class="bg-udmercy-blue mt-6 text-white font-text shadow-md font-medium px-5 text-sm py-3 rounded-full cursor-pointer"
        @click="() => userHasAddedCourse(course_to_add)"
      >
        Add Course
      </button>
    </Dialog>
  </main>
</template>

<script lang="ts" setup>
import { computed, ref } from 'vue'
import FileUpload from 'primevue/fileupload'
import { Dialog } from 'primevue'
import { useToast, TYPE } from 'vue-toastification'
import CancelIcon from '../assets/cancel-icon.png'
import axios from 'axios'
// @ts-ignore
import VueCal from 'vue-cal'

const degree_evaluation_was_uploaded = ref<boolean>(false)
const course_history_verified = ref<boolean>(false)
const requirements_satisfied = ref<any>([])
const requirements_not_satisfied = ref<any>([])
const processed_requirements = ref()
const course_info_is_visible = ref<boolean>(false)
const toast = useToast()
const uploading_file = ref<boolean>(false)
const fileupload = ref()
const events = ref<any>([])
const hide_courses_not_offered = ref<boolean>(false)
const chosen_courses = ref<any>([])
const course_prerequisites = ref<any>([])
const course_corequisites = ref<any>([])
const course_to_add = ref({})

const upload_url = computed(() => {
  return `${import.meta.env.VITE_API_URL}/upload_degree_evaluation`
})

const upload = () => {
  uploading_file.value = true
  fileupload.value.upload()
}

function formatCourseAttributes(course: any) {
  const attrs: any = []
  course.attributes.forEach((attr: any) => {
    attrs.push(attr.code.replace('KA', ''))
  })

  return attrs.join(' | ')
}

function onUpload(event: any) {
  const response = JSON.parse(event.xhr.response)
  requirements_satisfied.value = response.requirements_satisfied
  requirements_not_satisfied.value = response.requirements_not_satisfied
  degree_evaluation_was_uploaded.value = true
  console.log(response)
}

function removeSatisfiedReq(req_param: any) {
  requirements_satisfied.value = requirements_satisfied.value.filter(
    (req: any) => !(req.course === req_param.course && req.grade === req_param.grade),
  )
}

function formatCourseDays(course: any) {
  const days = []
  if (course.monday) {
    days.push('M')
  }
  if (course.tuesday) {
    days.push('T')
  }
  if (course.wednesday) {
    days.push('W')
  }
  if (course.thursday) {
    days.push('Th')
  }
  if (course.friday) {
    days.push('F')
  }
  if (course.saturday) {
    days.push('Sa')
  }
  if (course.sunday) {
    days.push('Sn')
  }

  return days.join(' | ')
}

async function addCourse(course: any) {
  try {
    if (course.is_selected) {
      // Deselect the course and remove it from the chosen courses list if it
      course.is_selected = false
      // @ts-ignore
      chosen_courses.value = chosen_courses.value.filter(
        (coursed: any) => coursed.course_id !== course.course_id,
      )
      events.value = events.value.filter((classs: any) => classs.course_id !== course.course_id)
      return
    }

    toast.clear()
    toast('Verifying if you can take the course...', {
      type: TYPE.INFO,
      timeout: 10000,
    })

    let res = await axios.post(`${import.meta.env.VITE_API_URL}/check_course_validity`, {
      course: course,
      requirements_satisfied: requirements_satisfied.value,
    })

    course_corequisites.value = res.data.corequisites
    course_prerequisites.value = res.data.prerequisites
    course_info_is_visible.value = true

    course_to_add.value = course
  } catch (err) {
    console.log(err)
    alert(err)
  }
}

function userHasAddedCourse(course: any) {
  course.is_selected = true
  chosen_courses.value.push(course)

  let enddate = course.start_date.split('/')
  let formattedEndDate = enddate[2] + '-' + enddate[1] + '-' + enddate[0]
  let startdate = course.end_date.split('/')
  let formattedStartDate = startdate[2] + '-' + startdate[1] + '-' + startdate[0]

  if (course.meeting_begin_time) {
    let beginTimeC = course.meeting_begin_time
    let begintime = beginTimeC.slice(0, 2) + ':' + beginTimeC.slice(2)

    let endTimeC = course.meeting_end_time
    let endtime = endTimeC.slice(0, 2) + ':' + endTimeC.slice(2)

    let days = []

    if (course.monday) {
      days.push('05')
    }
    if (course.tuesday) {
      days.push('06')
    }
    if (course.wednesday) {
      days.push('07')
    }
    if (course.thursday) {
      days.push('08')
    }
    if (course.friday) {
      days.push('09')
    }
    if (course.saturday) {
      days.push('10')
    }
    if (course.sunday) {
      days.push('11')
    }

    days.forEach((day) => {
      let starttime = '2025-05-' + day + ' ' + begintime
      let endtimes = '2025-05-' + day + ' ' + endtime

      events.value.push({
        start: starttime,
        end: endtimes,
        title: course.course_name,
        // content: `<p>${course.building}</p>`,
        class: 'health',
        course_id: course.course_id,
      })
    })
  }
  course_to_add.value = {}
  course_info_is_visible.value = false
}

async function startAdvisorAssistant() {
  try {
    course_history_verified.value = true

    let res = await axios.post(`${import.meta.env.VITE_API_URL}/start-assistant`, {
      requirements_not_satisfied: requirements_not_satisfied.value,
      requirements_satisfied: requirements_satisfied.value,
    })

    processed_requirements.value = res.data
  } catch (err) {}
}

function checkForSatisfiedReq(subject: any, course_number: any) {
  const course_mappings = {
    ACC: 'Accounting',
    ADS: 'Addiction Studies',
    AEV: 'Advanced Electric Vehicle',
    AAS: 'African-American Studies',
    ALCP: 'American Language & Culture',
    ANE: 'Anesthesiology',
    ARB: 'Arabic',
    AENG: 'Architectural Engineering',
    ARCH: 'Architecture',
    BIO: 'Biology',
    BUS: 'Business Administration',
    CCPD: 'Career & Prof Development',
    CAS: 'Catholic Studies',
    CHM: 'Chemistry',
    CHI: 'Chinese',
    CIVE: 'Civil Engineering',
    CST: 'Communication Studies',
    COM: 'Community Dentistry',
    MCD: 'Community Development',
    CSSE: 'Comp Sci/Software Engineering',
    CIS: 'Computer & Information Systems',
    CNS: 'Counseling',
    CJS: 'Criminal Justice',
    CYBE: 'Cybersecurity',
    DATA: 'Data Analytics',
    DENT: 'Dental General',
    ECN: 'Economics',
    ELEE: 'Electrical Engineering',
    ENGR: 'Engineering',
    CTA: 'Engineering Co-op',
    ENL: 'English',
    ETHL: 'Ethical Leadership',
    ETH: 'Ethics',
    FINA: 'Fine Arts',
    FRE: 'French',
    GEO: 'Geography',
    GER: 'German',
    GRA: 'Graduate Assistant',
    HLH: 'Health Professions',
    HSA: 'Health Services Administration',
    HIS: 'History',
    HON: 'Honors',
    INT: 'Intelligence Analysis',
    ISLM: 'Islamic Studies',
    JPN: 'Japanese',
    KOR: 'Korean',
    LAT: 'Latin',
    LAW: 'Law',
    LEAD: 'Leadership',
    LST: 'Legal Studies',
    MLS: 'Liberal Studies',
    MBA: 'MBA',
    MTH: 'Mathematics',
    MENG: 'Mechanical Engineering',
    MUSM: 'Museum Studies',
    MUS: 'Music',
    NUR: 'Nursing',
    PHL: 'Philosophy',
    PAS: 'Physician Assistant',
    PHY: 'Physics',
    PLS: 'Polish',
    POL: 'Political Science',
    MPD: 'Product Development',
    PYC: 'Psychology',
    RELS: 'Religious Studies',
    SCIE: 'Science',
    SWK: 'Social Work',
    SOC: 'Sociology',
    SPA: 'Spanish',
    STA: 'Statistics',
    TRE: 'Theatre',
    UAS: 'University Academic Services',
    VCE: 'Vehicle Cyber Engineering',
    WGS: "Women's & Gender Studies",
  }

  const reverse_course_mappings = Object.fromEntries(
    Object.entries(course_mappings).map(([key, value]) => [value, key]),
  )

  const subject_title = reverse_course_mappings[subject]
  console.log(subject_title)

  if (subject_title === undefined) {
    return "Can't verify"
  }

  let grade_gotten = ''
  requirements_satisfied.value.forEach((req: any) => {
    let course_name = `${subject_title} ${course_number}`

    // console.log(course_name, req.course)
    if (req.course === course_name) {
      grade_gotten = req.grade
    }
  })

  if (grade_gotten === '') {
    return 'Not satisfied'
  }

  return `Satisfied with a grade of ${grade_gotten}`
}

function saveAsTxtFile() {
  let final_course_content = ''
  let total_credits = 0
  chosen_courses.value.forEach((course: any) => {
    final_course_content += `Course Name: ${course.course_name} ${course.course_number}
Time: ${formatCourseTime(course)}
Credits: ${course.credits}
Section: ${course.section}
 
`

    total_credits += course.credits
  })

  final_course_content += `Total Credits: ${total_credits}`

  const blob = new Blob([final_course_content], { type: 'text/plain' })
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = 'course_selection.txt' // Set file name

  // Trigger download
  document.body.appendChild(link)
  link.click()

  // Cleanup
  document.body.removeChild(link)
  URL.revokeObjectURL(link.href)
}

function formatCourseTime(course: any) {
  if (!course.meeting_begin_time) {
    return 'Course has no specified meeting time'
  }

  let beginTimeC = course.meeting_begin_time
  let begintime = beginTimeC.slice(0, 2) + ':' + beginTimeC.slice(2)

  let endTimeC = course.meeting_end_time
  let endtime = endTimeC.slice(0, 2) + ':' + endTimeC.slice(2)

  return `${begintime} - ${endtime}`
}
</script>

<style>
:root {
  --udmercy-red: #a5093e;
  --udmercy-blue: #002d72;
}

.Vue-Toastification__toast {
  font-family: 'Poppins';
}

.Vue-Toastification__toast--info {
  border-radius: 0px;
  font-size: 4px;
  background-color: white;
  color: black;
  border-left: solid 7px black;
  box-shadow:
    0 1px 10px 0 rgba(111, 111, 111, 0.1),
    0 2px 15px 0 rgba(161, 161, 161, 0.05);
}

.Vue-Toastification__toast--error {
  border-radius: 0px;
  font-size: 4px;
  background-color: white;
  color: var(--udmercy-red);
  border-left: solid 7px var(--udmercy-red);
  box-shadow:
    0 1px 10px 0 rgba(111, 111, 111, 0.1),
    0 2px 15px 0 rgba(161, 161, 161, 0.05);
}

.Vue-Toastification__toast--warning {
  border-radius: 0px;
  font-size: 4px;
  background-color: white;
  color: #c78204;
  border-left: solid 7px #c78204;
  box-shadow:
    0 1px 10px 0 rgba(111, 111, 111, 0.1),
    0 2px 15px 0 rgba(161, 161, 161, 0.05);
}

.Vue-Toastification__toast--success {
  border-radius: 0px;
  font-size: 4px;
  background-color: white;
  color: var(--udmercy-blue);
  border-left: solid 7px var(--udmercy-blue);
  box-shadow:
    0 1px 10px 0 rgba(111, 111, 111, 0.1),
    0 2px 15px 0 rgba(161, 161, 161, 0.05);
}

.Vue-Toastification__toast--info .Vue-Toastification__close-button {
  color: black;
}

.Vue-Toastification__toast--info .Vue-Toastification__progress-bar {
  background-color: black;
}

.Vue-Toastification__toast--error .Vue-Toastification__close-button {
  color: var(--udmercy-red);
}

.Vue-Toastification__toast--success .Vue-Toastification__close-button {
  color: var(--udmercy-blue);
}
.Vue-Toastification__toast--warning .Vue-Toastification__close-button {
  color: #c78204;
}

.Vue-Toastification__toast--error .Vue-Toastification__progress-bar {
  background-color: var(--udmercy-red);
}
.Vue-Toastification__toast--success .Vue-Toastification__progress-bar {
  background-color: var(--udmercy-blue);
}
.Vue-Toastification__toast--warning .Vue-Toastification__progress-bar {
  background-color: #c78204;
}
.Vue-Toastification__toast-body {
  font-size: 13.5px;
}
</style>
