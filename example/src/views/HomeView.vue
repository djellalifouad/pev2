<script lang="ts" setup>
import { inject, ref, onMounted } from "vue"
import axios from "axios"
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome"
import { library } from "@fortawesome/fontawesome-svg-core"
import { fas } from "@fortawesome/free-solid-svg-icons"
library.add(fas)

import { time_ago } from "../utils"
import MainLayout from "../layouts/MainLayout.vue"
import Plan from "@/components/Plan.vue"
import {
  plan1_source,
  plan1_source_json,
  plan1_query,
  plan2_source,
  plan2_query,
  plan3_source,
  plan3_query,
  plan4_source,
  plan5_source,
  plan5_query,
  plan6_source,
  plan7_source,
  plan7_query,
  plan8_source,
  plan9_source,
  plan9_query,
  plan_parallel_source,
  plan_parallel_2_source,
  plan_parallel_2_query,
  plan_trigger_source,
  plan_trigger_query,
  plan_trigger_2_source,
  plan_trigger_2_query,
} from "../samples.ts"

import idb from "../idb"

const setPlanData = inject("setPlanData")

const planInput = ref<string>("")
const queryInput = ref<string>("")
const queryName = ref<string>("")
const draggingPlan = ref<boolean>(false)
const draggingQuery = ref<boolean>(false)
const savedPlans = ref<Plan[]>()
interface Sample extends Array<string> {
  0: string
  1: string
  2: string
}

const samples = ref<Sample[]>([
  ["Example 1 TEXT", plan1_source, plan1_query],
  ["Example 1 JSON", plan1_source_json, plan1_query],
  ["Example 2", plan2_source, plan2_query],
  ["Example 3", plan3_source, plan3_query],
  ["Example 5", plan5_source, plan5_query],
  ["With subplan", plan6_source, ""],
  ["With Buffers", plan7_source, plan7_query],
  ["With CTE", plan9_source, plan9_query],
  ["With CTEs", plan4_source, ""],
  ["Very large plan", plan8_source, ""],
  ["With trigger", plan_trigger_2_source, plan_trigger_2_query],
  ["With trigger (plain text)", plan_trigger_source, plan_trigger_query],
  ["Parallel (verbose)", plan_parallel_source, ""],
  ["Parallel (4 workers)", plan_parallel_2_source, plan_parallel_2_query],
])

function submitPlan() {
  let newPlan: Plan = ["", "", ""]
  newPlan[0] =
    queryName.value ||
    "New Plan - " +
      new Date().toLocaleString("en-US", {
        dateStyle: "medium",
        timeStyle: "medium",
      })
  newPlan[1] = planInput.value
  newPlan[2] = queryInput.value
  newPlan[3] = new Date().toISOString()
  savePlanData(newPlan)
  setPlanData(...newPlan)
}
async function savePlanData(sample: Plan) {
  await idb.savePlan(sample)
}
onMounted(() => {
  const textAreas = document.getElementsByTagName("textarea")
  Array.prototype.forEach.call(textAreas, (elem: HTMLInputElement) => {
    elem.placeholder = elem.placeholder.replace(/\\n/g, "\n")
  })
  const noHashURL = window.location.href.replace(/#.*$/, "")
  window.history.replaceState("", document.title, noHashURL)
  loadPlans()
})
async function loadPlans() {
  axios.get("http://127.0.0.1:8000/queries").then((response) => {
    var plans_not_final = response.data
    var plans_final = []
    for (var i in plans_not_final) {
      plans_final.push([
        plans_not_final[i].query,
        [
          JSON.stringify(plans_not_final[i].json_plan_pg_real),
          JSON.stringify(plans_not_final[i].json_plan_hinter_real),
          JSON.stringify(plans_not_final[i].json_plan_pg),
          JSON.stringify(plans_not_final[i].json_plan_hinter),
        ],
        "",
        "",
        plans_not_final[i].choosed_plan,
        plans_not_final[i].converge,
        plans_not_final[i].execution_time_pg,
        plans_not_final[i].execution_energy_pg,
        plans_not_final[i].execution_time_hybride,
        plans_not_final[i].execution_energy_hybrid,
        plans_not_final[i].id,
        plans_not_final[i].converge,
        plans_not_final[i].prefix_algo,
        plans_not_final[i].prefix,
      ])
    }
    savedPlans.value = plans_final.slice().reverse()
  })
}
function loadPlan(plan?: Plan) {
  if (!plan) {
    return
  }

  queryName.value = plan[0]
  planInput.value = plan[1]
  queryInput.value = plan[2]
}
function openPlan(plan: Plan, index: number) {
  setPlanData(plan[0], plan[1][index], plan[2])
}
function editPlan(plan: Plan) {
  loadPlan(plan)
}
async function deletePlan(plan: Plan) {
  await idb.deletePlan(plan)
  loadPlans()
}
function handleDrop(event: DragEvent) {
  const input = event.srcElement
  if (!(input instanceof HTMLTextAreaElement)) {
    return
  }
  draggingPlan.value = false
  draggingQuery.value = false
  if (!event.dataTransfer) {
    return
  }
  const file = event.dataTransfer.files[0]
  const reader = new FileReader()
  reader.onload = () => {
    if (reader.result instanceof ArrayBuffer) {
      return
    }
    input.value = reader.result || ""
    input.dispatchEvent(new Event("input"))
  }
  reader.readAsText(file)
}
</script>

<template>
  <main-layout>
    <div class="container">
      <div class="row">
        <div class="col d-flex"></div>
      </div>
      <div class="row">
        <div class="col-sm-7"></div>
        <div class="col-sm-12">
          <ul class="list-group" v-cloak>
            <li
              class="list-group-item px-2 py-1"
              v-for="plan in savedPlans"
              :key="plan.id"
            >
              <div class="row">
                <div class="col">
                  <a
                    v-on:click.prevent="openPlan(plan, 0)"
                    href=""
                    title="Open the plan details"
                  >
                    pg plan
                  </a>
                  /
                  <a
                    v-on:click.prevent="openPlan(plan, 1)"
                    href=""
                    title="Open the plan details"
                  >
                    hinter plan
                  </a>
                  /
                  <a
                    v-on:click.prevent="openPlan(plan, 2)"
                    href=""
                    title="Open the plan details"
                  >
                    pg plan estimated
                  </a>
                  /
                  <a
                    v-on:click.prevent="openPlan(plan, 3)"
                    href=""
                    title="Open the plan details"
                  >
                    hinter plan estimated
                  </a>
                </div>
              </div>
              <div class="row">
                <div class="col">
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      choosed plan {{ plan[4] }}
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Execution time pg {{ plan[6] }} Secondes
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Execution energy pg {{ plan[7] }} Joules
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Execution time hybride {{ plan[8] }} Secondes
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Execution energy hinter {{ plan[9] }} Joules
                    </span>
                  </small>
                  <br />
                  <a
                    v-on:click.prevent=""
                    href=""
                    title="Open the plan details"
                  >
                    Afficher les selections
                  </a>
                  <br />
                  <a
                    v-on:click.prevent=""
                    href=""
                    title="Open the plan details"
                  >
                    Afficher les projections
                  </a>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Converge {{ plan[11] }}
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Préfix algorithme {{ plan[12] }}
                    </span>
                  </small>
                  <br />
                  <small class="text-muted">
                    <span :title="plan[3]?.toString()">
                      Préfix {{ plan[13] }}
                    </span>
                  </small>
                </div>
                <div class="col-6 text-right">Query : {{ plan[0] }}</div>
                <div class="col-12 text-right">id : {{ plan[10] }}</div>
              </div>
            </li>
          </ul>
          <p class="text-muted text-center" v-if="!savedPlans?.length" v-cloak>
            <em> You haven't saved any plan yet.</em>
          </p>
        </div>
      </div>
    </div>
  </main-layout>
</template>
