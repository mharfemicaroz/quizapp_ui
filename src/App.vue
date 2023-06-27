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
        <div class="card-lg" v-if="!showAnswers">
          <div class="card-body">
            <h4 class="card-title question" style="margin-bottom: 60px;">{{ currentQuestion.question }}</h4>
            <div class="choices-container">
              <button v-for="choice in shuffledChoices" :key="choice" @click="selectChoice(choice)"
                class="btn btn-primary" :disabled="quizFinished">
                {{ choice }}
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
            <button @click="initializeQuiz" class="btn btn-primary">Restart Quiz</button>
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
                <td>{{ question.question }}</td>
                <td>{{ selectedChoices[index] }}</td>
                <td>{{ question.answerKey }}</td>
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
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Swal from 'sweetalert2';
import axios from 'axios';
import userdata from './assets/userdata.json'

export default {
  data() {
    return {
      loading: false,
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
          subcategories: ['Assessment of Learning', 'Child Adolescent & Learning Principles','Classroom Management','Curriculum Development','Principles of Teaching','Educational Technology','Facilitating Learning','Teaching Profession','Social Dimension','Field Study & Teaching Internship'],
        },
        {
          name: 'Specialization',
          subcategories: [],
        },
      ],
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
    quizzerName() {
      const matchedUser = this.userdata.find(
        (user) => user.email === this.login.username && user.password === this.login.password
      );

      if (matchedUser) {
        return matchedUser.username;
      }

      return '';
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
  },
  created() {
    this.userdata = userdata;
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
    selectCategory(category) {
      this.selectedCategory = category;
    },
    selectSubcategory(subcategory) {
      this.selectedSubcategory = subcategory;

      // Load quiz data based on selected subcategory
      switch (this.selectedSubcategory) {
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
        default:
          break;
      }
    },
    initializeQuiz(data) {
      this.questions = this.shuffleArray(data).slice(0, 10); // Limit to 10 questions
      this.currentQuestionIndex = 0;
      this.selectedChoices = [];
      this.quizFinished = false;
      this.showAnswers = false;
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
      document.body.addEventListener('contextmenu', this.handleContextMenu);
    });
  }

};
</script>


<style>
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
</style>
