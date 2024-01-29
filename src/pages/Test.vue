<template>
  <section class="reports">
    <div class="container">
      <div class="reports__inner">
        <div class="reports__filters">
          <div class="date__filter">
            <p>Выберите период</p>
            <date-picker
              v-model="value"
              type="date"
              range
              placeholder="Select datetime range"
              @change="changeDate($event)"
            ></date-picker>
          </div>
          <div class="select__filter">
            <multiselect
              v-model="selectEmployee"
              :options="employeesCount"
              :multiple="true"
              label="NAME"
              track-by="NAME"
              @input="createReport"
            ></multiselect>
          </div>
        </div>
        <table>
          <thead>
            <th>ФИО</th>
            <th>Задач всего в работе</th>
            <th>Поставлено задач</th>
            <th>Закрыто новых</th>
            <th>Закрыто все</th>
            <th>Время завершения(среднее)</th>
          </thead>
          <tbody>
            <tr v-for="employee in completedTasks" :key="employee.id">
              <td @click="fetchModalTasks(employee.id)">{{ employee.name }}</td>
              <td @click="fetchModalTasks(employee.id)">
                {{ employee.allTasks }}
              </td>
              <td @click="fetchModalTasks(employee.id)">
                {{ employee.completedTasks }}
              </td>
              <td @click="fetchModalTasks(employee.id)">
                {{ employee.closedNewTasks }}
              </td>
              <td @click="fetchModalTasks(employee.id)">
                {{ employee.closedAllTasks }}
              </td>
              <td
                @click="fetchModalTasks(employee.id)"
                v-if="employee.closedAllTasks != 0"
              >
                {{ converMSToDays(employee.averageTime) }}
              </td>
            </tr>
          </tbody>
        </table>
        <div
          @click="activeModalWindow()"
          class="modal"
          :class="{ activeModal: activeModal }"
        ></div>
        <div class="modal__inner" :class="{ activeWindow: activeWindow }">
          <table>
            <thead>
              <th>ID</th>
              <th>Заголовок</th>
              <th>Крайний срок</th>
              <th>Постановщик</th>
              <th>Ответственный</th>
              <th>Соисполнитель</th>
              <th>Наблюдатели</th>
              <th>Дата создания</th>
              <th>Дата закрытия</th>
            </thead>
            <tbody>
              <tr v-for="modalTask in modalTasks" :key="modalTask.id">
                <td>{{ modalTask.id }}</td>
                <td>{{ modalTask.title }}</td>
                <td>{{ modalTask.deadline }}</td>
                <td>{{ modalTask.createdBy }}</td>
                <td>{{ modalTask.responsibleId }}</td>
                <td>{{ modalTask.accomplices }}</td>
                <td>{{ modalTask.auditors }}</td>
                <td>{{ modalTask.createdDate }}</td>
                <td>{{ modalTask.closedDate }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import DatePicker from "vue2-datepicker";
import Multiselect from "vue-multiselect";
import "vue2-datepicker/index.css";
import "vue2-datepicker/locale/zh-cn";
import moment from "moment";
export default {
  name: "Test",
  components: { DatePicker, Multiselect },
  data: () => {
    return {
      value: [],
      employeesCount: [],
      completedTasks: [],
      modalTasks: [],
      selectEmployee: "",
      activeModal: false,
      activeWindow: false,
    };
  },

  methods: {
    changeDate(event) {
      this.value1 = event;
      this.createReport();
    },

    activeModalWindow() {
      this.activeModal = !this.activeModal;
      this.activeWindow = !this.activeWindow;
    },

    async fetchModalTasks(id) {
      let tasks = [];
      this.activeModalWindow();
      tasks = await this.fetchTasks();
      let obj = {};
      tasks.forEach((task) => {
        if (task.id == id) {
          obj.id = task.id;
          obj.title = task.title;
          obj.deadline = task.deadline;
          obj.createdBy = task.createdBy;
          obj.responsibleId = task.responsibleId;
          obj.accomplices = task.accomplices.join(", ");
          obj.auditors = task.auditors;
          obj.createdDate = moment(task.createdDate).format();
          obj.closedDate = moment(task.closedDate).format();
        }
      });
      this.modalTasks.push(obj);
    },

    converMSToDays(ms) {
      let sec = ms / 1000;
      let mins = sec / 60;
      let hours = mins / 60;

      let d = Math.floor(hours / 24);
      let h = Math.floor(hours % 24);
      let m = Math.floor(mins % 60);

      var dDisplay = d > 0 ? d + "д. " : "";
      var hDisplay = h > 0 ? h + "ч. " : "";
      var mDisplay = m > 0 ? m + "м. " : "";

      if (!d && !h && !m) {
        return 0;
      }
      return dDisplay + hDisplay + mDisplay;
    },

    getTimeTasks(tasks) {
      if (!tasks || !tasks.length) {
        return 0;
      }
      let ms = 0;
      tasks.forEach((task) => {
        if (task.status == "5") {
          ms += this.moment(task.closedDate).diff(
            this.moment(task.createdDate)
          );
        }
      });

      return ms / tasks.length;
    },

    async createReport() {
      let tasks = [];
      let filter = Object.assign({
        RESPONSIBLE_ID: this.selectEmployee.length
          ? this.selectEmployee.map((item) => item.ID)
          : [],
      });
      tasks = await this.fetchTasks(filter);
      console.log(tasks);
      filter = Object.assign(
        {
          ">=CLOSED_DATE": moment(this.value[0]).format(),
          "<=CLOSED_DATE": moment(this.value[1]).format(),
        },
        {
          RESPONSIBLE_ID: this.selectEmployee.length
            ? this.selectEmployee.map((item) => item.ID)
            : [],
        }
      );
      console.log(filter);
      let closedDate = await this.fetchTasks(filter);

      filter = Object.assign(
        {
          ">=CREATED_DATE": moment(this.value[0]).format(),
          "<=CREATED_DATE": moment(this.value[1]).format(),
        },
        {
          RESPONSIBLE_ID: this.selectEmployee.length
            ? this.selectEmployee.map((item) => item.ID)
            : [],
        }
      );

      let createDate = await this.fetchTasks(filter);

      this.completedTasks = [];

      this.employeesCount.forEach((employee) => {
        let obj = {
          id: employee.ID,
          name: employee.NAME,
        };

        let employeeTasks = tasks.filter(
          (task) => task.status != "5" && task.responsible.id == employee.ID
        );

        let completedTasks = tasks.filter(
          (completedTask) =>
            completedTask.status == "5" &&
            completedTask.responsible.id == employee.ID
        );

        let closedNewTasks = createDate.filter(
          (task) => task.responsible.id == employee.ID
        );

        let closedAllTasks = closedDate.filter(
          (task) => task.responsible.id == employee.ID
        );

        let userTime = this.getTimeTasks(closedAllTasks);

        obj.allTasks = employeeTasks.length;
        obj.completedTasks = completedTasks.length;
        obj.closedNewTasks = closedNewTasks.length;
        obj.closedAllTasks = closedAllTasks.length;
        obj.averageTime = userTime;

        this.completedTasks.push(obj);
      });
    },

    async fetchTasks(filter) {
      let tasks = [];
      tasks = (
        await this.callMethod("tasks.task.list", {
          filter: {
            ...filter,
          },
          select: [
            "ID",
            "TITLE",
            "STATUS",
            "GROUP_ID",
            "CREATED_BY",
            "CREATED_DATE",
            "RESPONSIBLE_ID",
            "ACCOMPLICES",
            "AUDITORS",
            "CHANGED_BY",
            "CHANGED_DATE",
            "CLOSED_DATE",
            "ACTIVITY_DATE",
            "DATE_START",
            "DEADLINE",
            "MARK",
            // '*',
          ],
        })
      )
        .map((el) => (el["tasks"] ? el["tasks"] : el))
        .flat();
      return tasks;
    },
  },

  async mounted() {
    this.employeesCount = await this.callMethod("user.get", {
      FILTER: {
        ACTIVE: true,
        USER_TYPE: "employee",
      },
    });
    await this.createReport();
  },
};
</script>

<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>

<style>
.container {
  max-width: 1240px;
  padding: 0 20px;
  margin: 0 auto;
}

.reports__inner {
  width: 100%;
}

table {
  width: 100%;
}

.modal {
  background: rgba(0, 0, 0, 0.63);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  visibility: hidden;
  opacity: 0;
  transition: all 0.3s ease-in-out;
  z-index: -1;
}

.activeModal {
  z-index: 999;
  opacity: 1;
  visibility: visible;
  transition: all 0.3s ease-in-out;
}

.modal__inner {
  z-index: 1000;
  padding: 5px;
  width: 86%;
  height: 50%;
  background-color: #fff;
  position: fixed;
  top: 19%;
  right: 97px;
  visibility: hidden;
  opacity: 0;
  transition: all 0.3s ease-in-out;
  border-radius: 20px;
}

.activeWindow {
  z-index: 1000;
  opacity: 1;
  visibility: visible;
  transition: all 0.3s ease-in-out;
}

.reports__filters {
  display: flex;
  flex-direction: column;
  gap: 30px 0;
}

.date__filter p {
  margin-bottom: 15px;
}

.reports__inner table {
  margin-top: 15px;
  border-collapse: collapse;
}

.reports__inner table tbody tr td {
  text-align: center;
  border: 2px solid #e8e8e8;
  padding: 5px 0;
}
</style>
