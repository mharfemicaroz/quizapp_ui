<template>
  <div class="container-lg">
    <div v-if="!loggedIn">
      <h2>Login</h2>
      <form @submit.prevent="loginUser">
        <div class="form-group">
          <label for="username">Username</label>
          <input type="email" id="username" v-model="login.username" class="form-control" required>
        </div>
        <div class="form-group">
          <label for="password">Password</label>
          <input type="password" id="password" v-model="login.password" class="form-control" required>
        </div>
        <button type="submit" class="btn btn-primary" style="margin-top: 10px;">Login</button>
      </form>
    </div>
    <div v-else>
      <div v-if="!selectedCategory">
        <h1>Welcome, {{ quizzerName }}</h1>
        <h2 style="margin-bottom: 25px;">Select a Category</h2>
        <div class="categories">
          <button v-for="category in categories" :key="category.name" @click="selectCategory(category)"
            class="btn btn-primary btn-lg category">
            {{ category.name }}
          </button>
          <button v-if="isAdmin" @click="selectAll" class="btn btn-primary btn-lg category">All</button>
        </div>
      </div>
      <div v-else-if="!selectedSubcategory">
        <div v-if="selectedCategory && selectedCategory.subcategories.length > 0">
          <h2 style="margin-bottom: 25px;">Select a Subcategory</h2>
          <div class="subcategories">
            <button v-for="subcategory in selectedCategory.subcategories" :key="subcategory"
              @click="selectSubcategory(subcategory)" class="btn btn-primary btn-lg category">
              {{ subcategory }}
            </button>
          </div>
        </div>
        <div v-else>
          <h2>No Subcategories Available</h2>
          <p style="margin-bottom: 25px;">Please choose a category again:</p>
          <div class="categories">
            <button v-for="category in categories" :key="category.name" @click="selectCategory(category)"
              class="btn btn-primary btn-lg category">
              {{ category.name }}
            </button>
          </div>
        </div>
      </div>
      <div v-else-if="loading" class="text-center mt-4">
        <img src="https://i.pinimg.com/originals/07/24/88/0724884440e8ddd0896ff557b75a222a.gif" width="600"
          height="600" />
      </div>
      <div v-else>
        <div v-if="isAdmin">
          <div v-if="isGroupDataLoading">
            <h2>Summary on {{ selectedCategory.name }} / {{ selectedSubcategory }}</h2>
            <table class="table mt-4">
              <thead>
                <tr>
                  <th>No.</th>
                  <th>username</th>
                  <th>attempts</th>
                  <th>average</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(item, index) in filteredGroupData" :key="index">
                  <td>{{ index + 1 }}</td>
                  <td>{{ item.username }}</td>
                  <td>{{ item.attempts }}</td>
                  <td>{{ item.avg }}</td>
                </tr>
              </tbody>
            </table>
            <button @click="initializeQuiz" class="btn btn-primary">Go Back</button>
          </div>
          <div v-else class="text-center mt-4">
            <img src="https://i.pinimg.com/originals/07/24/88/0724884440e8ddd0896ff557b75a222a.gif" width="600"
              height="600" />
          </div>
        </div>
        <div v-else>
          <div class="card-lg" v-if="!showAnswers">
            <div class="card-body">
              <h4 class="card-title question" style="margin-bottom: 60px;" v-html="currentQuestion.question"></h4>
              <div class="choices-container">
                <button v-for="choice in shuffledChoices" :key="choice" @click="selectChoice(choice)"
                  class="btn btn-primary" :disabled="quizFinished" v-html="choice">
                </button>
              </div>
            </div>
          </div>
          <div v-else>
            <h1>Congratulations {{ quizzerName }}!</h1>
            <h2>Quiz Result</h2>
            <div class="result">
              <p>You scored {{ correctAnswers }}/{{ questions.length }}</p>
              <p>Percentage: {{ percentage.toFixed(2) }}%</p>
              <p>You outperformed {{ getPercentile }}% of the quizzers in {{ selectedSubcategory }}.</p>
              <div class="row justify-content-center" style="margin-bottom: 25px;">
                <div class="col-md-12">
                  <div class="card">
                    <div class="card-header text-dark text-center">
                      <strong>Attempts</strong>
                    </div>
                    <div class="card-body chart">
                      <line-chart :chartData="lineData" />
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <table class="table mt-4">
              <thead>
                <tr>
                  <th>Question</th>
                  <th>Selected Choice</th>
                  <th>Correct Answer</th>
                  <th>Result</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(question, index) in questions" :key="index">
                  <td v-html="question.question"></td>
                  <td v-html="selectedChoices[index]"></td>
                  <td v-html="question.answerKey"></td>
                  <td>
                    <span v-if="selectedChoices[index] === question.answerKey" class="text-success">
                      <i class="bi bi-check-circle-fill"></i>
                    </span>
                    <span v-else class="text-danger">
                      <i class="bi bi-x-circle-fill"></i>
                    </span>
                  </td>
                </tr>
              </tbody>
            </table>
            <button @click="initializeQuiz" class="btn btn-primary">Restart Quiz</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import LineChart from "./components/LineChart.vue";
import Swal from 'sweetalert2';
import axios from 'axios';
import userdata from './assets/userdata.json'

export default {
  components: {
    LineChart
  },
  data() {
    return {
      isDone: false,
      loading: false,
      isGroupDataLoading: false,
      allIsClicked: false,
      quizzerdata: [],
      groupdata: [],
      alldata: [],
      login: {
        username: '',
        password: '',
      },
      loggedIn: false,
      categories: [
        {
          name: 'General Education',
          subcategories: ['English', 'Filipino', 'Science', 'Math', 'ICT', 'Social Studies'],
        },
        {
          name: 'Professional Education',
          subcategories: ['Assessment of Learning', 'Child Adolescent & Learning Principles', 'Classroom Management', 'Curriculum Development', 'Principles of Teaching', 'Educational Technology', 'Facilitating Learning', 'Teaching Profession', 'Social Dimension', 'Field Study & Teaching Internship', 'Current Trends in Education/Mix Topics'],
        },
        {
          name: 'Specialization',
          subcategories: ['Major-English', 'Major-Home Economics', 'Major-Mathematics'],
        },
      ],
      lineData: {
        labels: [],
        datasets: [
          {
            data: [],
          }
        ]
      },
      selectedCategory: null,
      selectedSubcategory: null,
      questions: [],
      currentQuestionIndex: 0,
      selectedChoices: [],
      quizFinished: false,
      showAnswers: false,
    };
  },
  computed: {
    isAdmin() {
      if (this.loggedIn && this.login.username === 'admin@knp-fc-ep.vercel.app') {
        return true;
      }
      return false;
    },
    quizzerName() {
      const matchedUser = this.userdata.find(
        (user) => user.email === this.login.username && user.password === this.login.password
      );

      if (matchedUser) {
        return matchedUser.username;
      }

      return '';
    },
    filteredGroupData() {
      const odata = (this.allIsClicked) ? this.alldata : this.groupdata;
      const uniqueUsernames = [...new Set(odata.map(item => item.username))];
      const resultArray = [];

      for (const username of uniqueUsernames) {
        const filteredData = odata.filter(item => item.username === username);
        const attempts = filteredData.length;
        const accumulatedPercentage = filteredData.reduce((sum, item) => sum + parseFloat(item.percentage), 0);
        const avg = (accumulatedPercentage / attempts).toFixed(2);

        resultArray.push({ username, percentage: accumulatedPercentage, avg, attempts });
      }

      // Sort the resultArray based on accumulatedPercentage in descending order
      resultArray.sort((a, b) => b.percentage - a.percentage);

      return resultArray;
    },
    currentQuestion() {
      return this.questions && this.questions[this.currentQuestionIndex] ? this.questions[this.currentQuestionIndex] : [];
    },
    shuffledChoices() {
      return this.shuffleArray([
        this.currentQuestion.choice1,
        this.currentQuestion.choice2,
        this.currentQuestion.choice3,
        this.currentQuestion.choice4,
      ]);
    },
    correctAnswers() {
      return this.questions.reduce((count, question, index) => {
        return count + (question.answerKey === this.selectedChoices[index] ? 1 : 0);
      }, 0);
    },
    percentage() {
      return (this.correctAnswers / this.questions.length) * 100;
    },
    quizzerAverage() {
      const n = this.quizzerdata.length;
      const sum = this.quizzerdata.reduce((accumulator, currentValue) => {
        return accumulator + parseFloat(currentValue.percentage);
      }, 0);
      const avg = (sum / n).toFixed(2);
      return avg;
    },
    groupAverage() {
      const n = this.groupdata.length;
      const sum = this.groupdata.reduce((accumulator, currentValue) => {
        return accumulator + parseFloat(currentValue.percentage);
      }, 0);
      const avg = (sum / n).toFixed(2);
      return avg;
    },
    groupStdev() {
      let arr = [];
      for (const item of this.quizzerdata) {
        arr.push(parseFloat(item.percentage));
      }
      return this.getStandardDeviation(arr);
    },
    getZscore() {
      return (this.quizzerAverage - this.groupAverage) / this.groupStdev;
    },
    getPercentile() {
      return (this.GetZPercent(this.getZscore) * 100).toFixed(2);
    },
  },
  created() {
    this.userdata = userdata;
  },
  watch: {
    currentQuestion(newQuestion, oldQuestion) {
      // Perform actions when the currentQuestion changes
      console.log('Current question changed:', newQuestion);
      // Add your custom logic here...

      // For example, you can check if the newQuestion is the last question
      if (this.currentQuestionIndex === this.questions.length - 1) {
        console.log('Last question reached!');
        // Perform any specific actions for the last question
      }

      this.$nextTick(() => {
        MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
      });
    },
  },
  methods: {
    loginUser() {
      const matchedUser = this.userdata.find(
        (user) =>
          user.email === this.login.username && user.password === this.login.password
      );

      if (matchedUser) {
        // User login successful
        this.loggedIn = true;
        this.selectedCategory = null;
        this.selectedSubcategory = null;
        Swal.fire({
          icon: 'success',
          title: 'Login Successful',
          text: 'You have successfully logged in.',
        });
      } else {
        // User login failed
        Swal.fire({
          icon: 'error',
          title: 'Login Failed',
          text: 'Username or password does not match our records.',
        });
      }

    },
    loaddata() {
      this.isDone = false;
      this.isGroupDataLoading = false;
      if (!this.allIsClicked) {
        axios
          .get(`https://api.knp.edu.ph/api/public/api/results?username=${encodeURIComponent(this.login.username)}&category=${encodeURIComponent(this.selectedCategory.name)}&subcategory=${encodeURIComponent(this.selectedSubcategory)}`)
          .then((response) => {
            this.quizzerdata = response.data.results;
          })
          .catch((error) => {
            console.error('Error loading the data:', error);
          })
          .finally(() => {
            this.isDone = true; // Set loading state to false when the request is completed
          });
        axios
          .get(`https://api.knp.edu.ph/api/public/api/results?category=${encodeURIComponent(this.selectedCategory.name)}&subcategory=${encodeURIComponent(this.selectedSubcategory)}`)
          .then((response) => {
            this.groupdata = response.data.results;
          })
          .catch((error) => {
            console.error('Error loading the data:', error);
          })
          .finally(() => {
            this.isGroupDataLoading = true;
          });
      } else {
        axios
          .get(`https://api.knp.edu.ph/api/public/api/results`)
          .then((response) => {
            this.alldata = response.data.results;
          })
          .catch((error) => {
            console.error('Error loading the data:', error);
          })
          .finally(() => {
            this.isGroupDataLoading = true;
          });
      }

    },
    getStandardDeviation(array) {
      const o = array || [];
      const n = o.length;

      if (n === 0) {
        return 0; // Return 0 for an empty array
      }

      const mean = o.reduce((a, b) => a + b, 0) / n;
      return Math.sqrt(o.map(x => Math.pow(x - mean, 2)).reduce((a, b) => a + b, 0) / n);
    },
    GetZPercent(z) {

      // z == number of standard deviations from the mean

      // if z is greater than 6.5 standard deviations from the mean the
      // number of significant digits will be outside of a reasonable range

      if (z < -6.5) {
        return 0.0;
      }

      if (z > 6.5) {
        return 1.0;
      }

      var factK = 1;
      var sum = 0;
      var term = 1;
      var k = 0;
      var loopStop = Math.exp(-23);

      while (Math.abs(term) > loopStop) {
        term = .3989422804 * Math.pow(-1, k) * Math.pow(z, k) / (2 * k + 1) / Math.pow(2, k) * Math.pow(z, k + 1) / factK;
        sum += term;
        k++;
        factK *= k;
      }

      sum += 0.5;

      return sum;
    },
    processQuizzerData() {
      this.quizzerdata.push({
        "id": "final",
        "percentage": this.percentage,
      });
      console.log(this.quizzerdata)
      for (const item of this.quizzerdata) {
        this.lineData.labels.push(item.id);
        this.lineData.datasets[0].data.push(parseFloat(item.percentage).toFixed(2));
      }
      this.loaded = Array(0).fill(true);
      MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
    },
    selectCategory(category) {
      this.selectedCategory = category;
    },
    selectAll() {
      this.allIsClicked = true;
      this.selectedCategory = {
        name: 'all',
        subcategories: [],
      };
      this.selectedSubcategory = 'all';
      this.loaddata();
    },
    selectSubcategory(subcategory) {
      this.selectedSubcategory = subcategory;

      // Load quiz data based on selected subcategory
      switch (this.selectedSubcategory) {
        case 'Major-Mathematics':
          import('./assets/quizData-majmth.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Major-Home Economics':
          import('./assets/quizData-majtle.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Major-English':
          import('./assets/quizData-majeng.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'English':
          import('./assets/quizData-english.json').then((module) => {
            this.initializeQuiz(module.default);

          });
          break;
        case 'Filipino':
          import('./assets/quizData-filipino.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Science':
          import('./assets/quizData-science.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Math':
          import('./assets/quizData-math.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'ICT':
          import('./assets/quizData-ict.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Social Studies':
          import('./assets/quizData-socsci.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Assessment of Learning':
          import('./assets/quizData-aol.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Child Adolescent & Learning Principles':
          import('./assets/quizData-cad.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Classroom Management':
          import('./assets/quizData-cm.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Curriculum Development':
          import('./assets/quizData-curdev.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Principles of Teaching':
          import('./assets/quizData-pt.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Educational Technology':
          import('./assets/quizData-edtech.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Facilitating Learning':
          import('./assets/quizData-fl.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Teaching Profession':
          import('./assets/quizData-tf.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Social Dimension':
          import('./assets/quizData-sd.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Field Study & Teaching Internship':
          import('./assets/quizData-fsti.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        case 'Current Trends in Education/Mix Topics':
          import('./assets/quizData-cte.json').then((module) => {
            this.initializeQuiz(module.default);
          });
          break;
        default:
          break;
      }
      this.loaddata();
    },
    shuffleArray(array) {
      // Fisher-Yates shuffle algorithm
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    selectChoice(choice) {
      this.selectedChoices.push(choice);
      if (this.currentQuestionIndex < this.questions.length - 1) {
        this.currentQuestionIndex++;
      } else {
        this.showResult();
      }
    },
    showResult() {
      this.loading = true; // Add a loading state
      this.quizFinished = true;
      const resultData = {
        username: this.login.username,
        datecreated: new Date().toISOString(),
        percentage: this.percentage,
        category: this.selectedCategory.name,
        subcategory: this.selectedSubcategory,
      };
      axios
        .post('https://api.knp.edu.ph/api/public/api/results', resultData)
        .then(() => {
          Swal.fire({
            title: 'Quiz Result',
            html: `You scored ${this.correctAnswers}/${this.questions.length}<br>Percentage: ${this.percentage.toFixed(
              2
            )}%`,
            icon: 'success',
            confirmButtonText: 'Ok',
          }).then(() => {
            this.showAnswers = true;
            MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
            this.processQuizzerData();
          });
        })
        .catch((error) => {
          console.error('Error saving quiz result:', error);
          Swal.fire({
            title: 'Error',
            text: 'An error occurred while saving the quiz result. Please try again.',
            icon: 'error',
            confirmButtonText: 'OK',
          });
        })
        .finally(() => {
          this.loading = false; // Set loading state to false when the request is completed
        });
    },
    initializeQuiz(data) {
      if (Array.isArray(data)) {
        this.questions = this.shuffleArray(data).slice(0, 10); // Limit to 10 questions
        this.currentQuestionIndex = 0;
        this.selectedChoices = [];
        this.quizFinished = false;
        this.showAnswers = false;
        this.lineData.labels = [];
        this.lineData.datasets[0].data = [];
        this.quizzerdata = [];
        this.isDone = false;
        this.isGroupDataLoading = false;
        this.isAllDataLoading = true;
        MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
      } else {
        this.selectedCategory = null;
        this.selectedSubcategory = null;
      }
    },
    handleContextMenu(event) {
      event.preventDefault();
    },
  },
  mounted() {
    this.$nextTick(() => {
      //document.body.addEventListener('contextmenu', this.handleContextMenu);
    });


  }

};
</script>


<style scoped>
.choices-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
}

.btn {
  width: 250px;
}

.category {
  margin-right: 10px;
  margin-bottom: 25px;
}

.question {
  margin-bottom: 25px;
}

.result {
  text-align: center;
  margin-bottom: 20px;
}

.text-success {
  color: #28a745;
}

.text-danger {
  color: #dc3545;
}

.card-header {
  border: none;
  background-color: transparent;
}

.card-body.chart {
  height: 300px;
}
</style>
