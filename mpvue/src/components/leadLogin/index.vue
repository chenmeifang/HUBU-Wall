<template>
  <div>
    <i-action-sheet :visible="showLoginSheet" show-cancel mask-closable="false">
      <view slot="header" style="padding: 16px">
        <i-button open-type="getUserInfo" @getuserinfo="getUserAndReg">获取用户信息(授权登录)</i-button>
      </view>
    </i-action-sheet>
  </div>
</template>

<script>
import { formatDatetime } from "../../utils/formatDatetime";

export default {
  data() {
    return {
      showLoginSheet: false
    };
  },
  methods: {
    /**
     * 登录方法
     * 逻辑：用code从服务端获取信息
     * 如果获取到的是openId，则执行新增用户逻辑
     * 如果获取到的是用户信息，则结束流程
     */
    login() {
      wx.login({
        success: res => {
          if (res.code) {
            this.$fly
              .get("/login/getOpenId", {
                code: res.code
              })
              .then(res => {
                if (typeof res.data === "string") {
                  this.showLoginSheet = true;
                  this.openId = res.data;
                } else {
                  this.$store.state.userInfo = res.data;
                  this.newWSConnect();
                  console.log(res.data);
                  wx.showToast({
                    icon: "none",
                    title: "登录成功,欢迎体验！",
                    duration: 2000
                  });
                }
              });
          }
        }
      });
    },
    /**
     * 从微信端获取用户信息
     * 将用户信息包装到promise中返回
     */
    getUserAndReg() {
      this.showLoginSheet = false;
      wx.getSetting({
        success: res => {
          // 判断之前有没有授权：之前授权过了
          if (res.authSetting["scope.userInfo"]) {
            wx.getUserInfo({
              success: res => {
                this.addUser(res.userInfo);
              }
            });
          } else {
            // 判断之前有没有授权：之前没有授权--->强制授权
            wx.authorize({
              scope: "scope.userInfo",
              success(res) {
                wx.getUserInfo({
                  success: res => {
                    this.addUser(res.userInfo);
                  }
                });
              },
              fail: err => {
                console.log(err);
              }
            });
          }
        },
        fail: err => {
          console.log(err);
        }
      });
    },
    /**
     * 拼接微信的getUserInfo和openId和学号等信息
     * 完成数据库用户注册
     */
    addUser(userInfo) {
      const date = new Date();
      const userNewLogin = formatDatetime(date);
      this.$fly
        .post("/login/newUser", {
          nick_name: userInfo.nickName,
          avatar: userInfo.avatarUrl,
          wx_id: this.openId,
          user_new_login: userNewLogin,
          phone_num: "",
          student_num: "",
          gender: userInfo.gender,
          city: userInfo.city,
          province: userInfo.province,
          socket_id: ""
        })
        .then(res => {
          this.$store.state.userInfo = {
            id: res.data.insertId,
            nick_name: userInfo.nickName,
            avatar: userInfo.avatarUrl,
            wx_id: this.openId,
            user_new_login: userNewLogin,
            gender: userInfo.gender,
            city: userInfo.city,
            province: userInfo.province,
          }
          if (res.data.affectedRows === 1) {
            this.newWSConnect();
            wx.showToast({
              icon: "none",
              title: "登录成功,欢迎体验！",
              duration: 2000
            });
          }
        });
    },
    newWSConnect() {
      this.$socket.emit("newConnect", {
        id: this.$store.state.userInfo.id
      });
    }
  },
  created() {
    this.login();
  }
};
</script>