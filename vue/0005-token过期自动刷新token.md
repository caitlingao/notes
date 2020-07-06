## token 过期自动刷新
```
// request.js

import axios from "axios";
import { Message, MessageBox } from "element-ui";
import store from "@/store";
import { getToken, getRefreshToken } from "@/utils/auth";

// 是否正在刷新的标记
let isRefreshing = false;
// 重试队列，每一项将是一个待执行的函数形式
let requests = [];

const baseURL = process.env.VUE_APP_BASE_API; // url = base url + request url
const headers = {
  Accept: "application/json",
  "Content-Type": "application/json",
  "Access-Control-Allow-Credentials": true,
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST",
  "Access-Control-Allow-Headers": "application/json"
};

// create an axios instance
const service = axios.create({
  baseURL: baseURL, // url = base url + request url
  // baseURL: "http://localhost:9000",
  withCredentials: true, // send cookies when cross-domain requests
  timeout: 5000 // request timeout
});

// request interceptor
service.interceptors.request.use(
  config => {
    config.headers = headers;
    if (store.getters.token) {
      config.headers["X-Token"] = getToken();
    }
    return config;
  },
  error => {
    // do something with request error
    console.log(error); // for debug
    return Promise.reject(error);
  }
);

// response interceptor
service.interceptors.response.use(
  response => {
    const res = response.data;
    if (res.code !== 0) {
      return Promise.reject(new Error(res.message || "Error"));
    } else {
      return res;
    }
  },
  error => {
    // Error对象可能log出来并不是你想象的那种以对象的样子出现
    console.log(error.response); // console.log(error) 401 再判断 error.response.data.code
    if (error.response.status === 401) {
      switch (error.response.data.code) {
        // 未传 token
        case 50008:
          Message({ message: "缺少 token", type: "error", duration: 3 * 1000 });
          return;
        case 50014: // token 过期自动更新
          return againRequest(error);
        case 50015: // refresh token 过期，return login
          console.log("refresh_token过期 超时......");
          resetLogin();
          return;
        default:
          Message({ message: "未授权，请重新登录(401)", type: "error" });
          resetLogin();
          return;
      }
    }

    if (error && error.message) {
      const errMsg = getErrMsg(error.response.status);
      error.message = errMsg;
    } else {
      error.message = "连接服务器失败!";
    }

    Message({ message: error.message, type: "error" });
    return Promise.reject(error);
  }
);

function againRequest(error) {
  const config = error.response.config;
  if (!isRefreshing) {
    isRefreshing = true;
    const oldRefreshToken = getRefreshToken();
    store
      .dispatch("user/refresh", { refreshToken: oldRefreshToken })
      .then(res => {
        const { token } = res.data;
        config.headers["YX-Token"] = token; // 赋值新的 token
        config.baseURL = ""; // url已经带上了/api，避免出现/api/api的情况
        // refresh token, retry all requests
        requests.forEach(cb => cb(token));
        requests = [];
        // retry current request and return promise
        return service(config);
      })
      .catch(res => {
        // refresh token failure, login
        console.error("refresh token error =>", res);
        resetLogin();
      })
      .finally(() => {
        isRefreshing = false;
      });
  } else {
    // freshing return and resolve promise
    return new Promise(resolve => {
      // put resolve into requests, action after refresh
      requests.push(token => {
        config.baseURL = "";
        config.headers["YX-Token"] = token; // 赋值新的 token
        resolve(service(config));
      });
    });
  }
}

function resetLogin() {
  store.dispatch("user/fedLogout").then(() => {
    location.reload(); // 为了重新实例化vue-router对象 避免bug
  });
}

function getErrMsg(resStatus) {
  let message;
  switch (resStatus) {
    case 400:
      message = "请求错误(400)";
      break;
    case 401:
      message = "未授权，请重新登录(401)";
      break;
    case 403:
      message = "拒绝访问(403)";
      break;
    case 404:
      message = "请求出错(404)";
      break;
    case 408:
      message = "请求超时(408)";
      break;
    case 500:
      message = "服务器错误(500)";
      break;
    case 501:
      message = "服务未实现(501)";
      break;
    case 502:
      message = "网络错误(502)";
      break;
    case 503:
      message = "服务不可用(503)";
      break;
    case 504:
      message = "网络超时(504)";
      break;
    case 505:
      message = "HTTP版本不受支持(505)";
      break;
    default:
      message = `连接出错)!`;
  }
  return message;
}

export default service;
```
```
// src/store/modules/user.js
import Vue from "vue";
import { getToken, getRefreshToken, getTokensExpiry, cleanCookies } from "@/utils/auth";
import router, { resetRouter } from "@/router";

const state = {
  token: getToken(),
  refreshToken: getRefreshToken(),
  tokensExpiry: getTokensExpiry(),
  roles: [],
};

const mutations = {
  SET_TOKEN: (state, token) => {
    state.token = token;
  },
  SET_REFRESH_TOKEN: (state, refreshToken) => {
    state.refreshToken = refreshToken;
  },
  SET_TOKENS_EXPIRY: (state, tokensExpiry) => {
    state.tokensExpiry = tokensExpiry;
  },
  SET_ROLES: (state, roles) => {
    state.roles = roles;
  },
};

const actions = {
  // 前端退出登录
  fedLogout({ commit, dispatch }) {
    return new Promise(resolve => {
      commit("SET_TOKEN", "");
      commit("SET_REFRESH_TOKEN", "");
      commit("SET_TOKENS_EXPIRY", "");
      commit("SET_ROLES", []);
      cleanCookies();
      Vue.ls.clear();
      resetRouter();

      resolve();
    });
  }
};

export default {
  namespaced: true,
  actions,
};

```
```
import Cookies from "js-cookie";

const TOKEN_KEY = "Admin-Token";
const REFRESH_TOKEN_KEY = "Admin-Refresh-Token";
const EXPIRES_IN = "expiresIn";

export function getToken() {
  return Cookies.get(TOKEN_KEY);
}

export function getRefreshToken() {
  return Cookies.get(REFRESH_TOKEN_KEY);
}

export function getTokensExpiry() {
  return Cookies.get(EXPIRES_IN);
}

export function cleanCookies() {
  Object.keys(Cookies.get()).forEach(function(cookieName) {
    Cookies.remove(cookieName);
  });
}
```
