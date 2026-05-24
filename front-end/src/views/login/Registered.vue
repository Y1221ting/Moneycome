<template>
    <div class="login-frame">
        <div class="login-name login-input">
            <input type="text" name="mobile" v-model="reg.mobile" autocomplete="off"/>
            <span class="placeholder" :class="{fixed: reg.mobile !== '' && reg.mobile != null}">输入手机号</span>
        </div>
        <div class="login-password login-input">
            <input type="password" name="password" v-model="reg.password" autocomplete="off"/>
            <span class="placeholder" :class="{fixed: reg.password !== '' && reg.password != null}">设置密码</span>
        </div>
        <div class="login-password login-input">
            <input type="password" name="confirmPassword" v-model="reg.confirmPassword" @keyup.enter="regSubmit" autocomplete="off"/>
            <span class="placeholder" :class="{fixed: reg.confirmPassword !== '' && reg.confirmPassword != null}">确认密码</span>
        </div>
        <div class="buttonDiv">
            <Button :loading="loading" block color="primary" size="l" @click="regSubmit">注册</Button>
        </div>
        <div class="margin" style="margin-bottom: 0 !important;">
            <span class="text-hover" @click="$emit('input','LoginForm')">返回登录</span>
        </div>
    </div>
</template>
<script>
    export default {
        name: 'Registered',
        data() {
            return {
                reg: {
                    mobile: "",
                    password: "",
                    confirmPassword: "",
                },
                loading: false
            };
        },
        methods: {
            regSubmit() {
                if (!this.reg.mobile) {
                    this.$Message.error('请输入手机号');
                    return;
                }
                if (!this.reg.password) {
                    this.$Message.error('请设置密码');
                    return;
                }
                if (this.reg.password !== this.reg.confirmPassword) {
                    this.$Message.error('两次密码不一致');
                    return;
                }
                this.loading = true;
                this.$api.common.registerSimple(this.reg).then((res) => {
                    this.$Message.success('注册成功，请登录');
                    this.$emit('input', 'LoginForm');
                }).finally(() => {
                    this.loading = false;
                });
            }
        }
    }
</script>
