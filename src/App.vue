<template>
  <h1>HW2 - Todo API</h1>
  <div>
    <h2>註冊</h2>
    <!-- {{ signUpData }} -->
    <input type="text" placeholder="Email" v-model="signUpData.email" />
    <input type="password" placeholder="Password" v-model="signUpData.password" />
    <input type="text" placeholder="Nickname" v-model="signUpData.nickname" />
    <button type="button" @click="signUp">註冊</button>
    <br />
    <p>{{ signUpRespMsg }}</p>
  </div>

  <div>
    <h2>登入</h2>
    <!-- {{ signInData }} -->
    <input type="text" placeholder="Email" v-model="signInData.email" />
    <input type="password" placeholder="Password" v-model="signInData.password" />
    <button type="button" @click="signIn">登入</button>
    <br />
    <p>{{ signInRespMsg }}</p>
  </div>

  <div>
    <h2>驗證</h2>
    <p>{{ checkoutRespMsg }}</p>
  </div>

  <div>
    <h2>登出</h2>
    <button type="button" @click="signOut">登出</button>
    <p>{{ signOutMsg }}</p>
  </div>
  <hr />

  <div v-if="LoginStatus">
    <h2>TODO</h2>
    <!-- {{ newTodo }} -->
    <input type="text" v-model="newTodo" placeholder="New Todo" />
    <button type="button" @click="addTodo">新增</button>
    {{ addTodoMsg }}
    <br />
    <!-- {{ todoList }} -->

    <table>
      <thead>
        <th>待辦事項</th>
        <th>狀態</th>
        <th>操作</th>
      </thead>
      <tbody>
        <tr v-for="todo in todoList" :key="todo.id">
          <td>{{ todo.content }}</td>
          <td>{{ todo.status ? '已完成' : '未完成' }}</td>
          <td>
            <button type="button" @click="toggleStatus(todo.id)">Toggle status</button>
            <button type="button" @click="delTodo(todo.id)">Delete</button>
            <button type="button" @click="updateTodo(todo.id)">Update</button>
            <input
              type="text"
              placeholder="newer value"
              v-model="TempNewValue[todo.id]"
              @change="saveTempNewValue($event, todo.id)"
            />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import axios from 'axios'
import { onMounted, ref } from 'vue'

const apiUrl = 'https://todolist-api.hexschool.io'

// 註冊
const signUpRespMsg = ref('')
const signUpData = ref({
  email: '',
  password: '',
  nickname: ''
})

const signUp = async () => {
  try {
    const response = await axios.post(`${apiUrl}/users/sign_up`, signUpData.value)
    signUpRespMsg.value = `uid: ${response.data.uid}`
    console.log(response)
  } catch (error) {
    signUpRespMsg.value = `errors: ${error.response.data.message}`
    console.log(error.response.data.message)
  }
}

// 登入
const signInRespMsg = ref('')
const signInData = ref({
  email: '',
  password: ''
})
const LoginStatus = ref(false)

const signIn = async () => {
  try {
    const response = await axios.post(`${apiUrl}/users/sign_in`, signInData.value)
    signInRespMsg.value = `token: ${response.data.token}`

    document.cookie = `todoToken=${response.data.token}; expires=${getExpDate(response.data.exp)}; path=/`
    console.log(response)

    validationLogin()
  } catch (error) {
    signInRespMsg.value = `errors: ${error.response.data.message}`
    console.log(error)
  }
}

const getExpDate = (expFromApi) => {
  // 將秒轉換為毫秒
  const expMilliseconds = expFromApi * 1000

  // 創建Date物件
  const expDate = new Date(expMilliseconds)

  // 獲取正確格式的expires字符串
  const expires = expDate.toUTCString()

  return expires
}

const todoToken = ref('')
// 驗證
const checkoutRespMsg = ref('')

const validationLogin = async () => {
  // 驗證
  try {
    todoToken.value = document.cookie.replace(
      /(?:(?:^|.*;\s*)todoToken\s*=\s*([^;]*).*$)|^.*$/,
      '$1'
    )
    const response = await axios.get(`${apiUrl}/users/checkout`, {
      headers: {
        Authorization: todoToken.value
      }
    })

    if (response.data.status) {
      checkoutRespMsg.value = `${response.data.nickname} : 已登入成功`
      LoginStatus.value = true
      getTodos()
    }
    console.log(response)
  } catch (error) {
    checkoutRespMsg.value = `errors: ${error.response.data.message}`
    console.log(error)
  }
}

onMounted(() => {
  validationLogin()
})

const signOutMsg = ref('')
// 登出
const signOut = async () => {
  try {
    todoToken.value = document.cookie.replace(
      /(?:(?:^|.*;\s*)todoToken\s*=\s*([^;]*).*$)|^.*$/,
      '$1'
    )

    const response = await axios.post(`${apiUrl}/users/sign_out`, null, {
      headers: {
        Authorization: todoToken.value
      }
    })
    // signInRespMsg.value = `token: ${response.data.token}`
    if (response.data.status) {
      signOutMsg.value = '登出成功'
      LoginStatus.value = false
    } else {
      signOutMsg.value = '登出失敗'
    }

    document.cookie = `todoToken=; max-age=0; path=/;`
    console.log(response)
  } catch (error) {
    signOutMsg.value = `errors: ${error.response.data.message}`
    console.log(error)
  }
}

// TODO
const newTodo = ref('')
const addTodoMsg = ref('')
const addTodo = async () => {
  try {
    const response = await axios.post(
      `${apiUrl}/todos/`,
      {
        content: newTodo.value
      },
      {
        headers: {
          Authorization: todoToken.value
        }
      }
    )

    if (response.status == 201) {
      addTodoMsg.value = `${response.data.newTodo.content} : 已新增成功`

      newTodo.value = ''
    }

    getTodos()

    console.log(response)
  } catch (error) {
    addTodoMsg.value = `errors: ${error.response.data.message}`
    console.log(error)
  }
}

const todoList = ref([])
const getTodos = async () => {
  try {
    const response = await axios.get(`${apiUrl}/todos/`, {
      headers: {
        Authorization: todoToken.value
      }
    })

    if (response.status == 200) {
      todoList.value = response.data.data
    }
    console.log(response)
  } catch (error) {
    console.log(error)
  }
}

onMounted(() => {
  getTodos()
})

// TODO : toggle
const toggleStatus = async (id) => {
  try {
    const response = await axios.patch(`${apiUrl}/todos/${id}/toggle`, null, {
      headers: {
        Authorization: todoToken.value
      }
    })

    getTodos()
    console.log(response)
  } catch (error) {
    console.log(error)
  }
}

// TODO : toggle
const delTodo = async (id) => {
  try {
    const response = await axios.delete(`${apiUrl}/todos/${id}`, {
      headers: {
        Authorization: todoToken.value
      }
    })

    getTodos()
    console.log(response)
  } catch (error) {
    console.log(error)
  }
}

// TODO : update
const TempNewValue = ref({})
const saveTempNewValue = (e, id) => {
  TempNewValue.value = {
    ...TempNewValue.value,
    [id]: e.target.value
  }
}

const updateTodo = async (id) => {
  const todo = todoList.value.find((t) => t.id == id)
  todo.content = TempNewValue.value[id]

  try {
    const response = await axios.put(`${apiUrl}/todos/${id}`, todo, {
      headers: {
        Authorization: todoToken.value
      }
    })

    delete TempNewValue.value[id]

    getTodos()
    console.log(response)
  } catch (error) {
    console.log(error)
  }
}
</script>
