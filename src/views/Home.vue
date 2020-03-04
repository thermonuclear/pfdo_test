<template lang="pug">
  .home
    img(alt="Game logo" src="../assets/logo.png")
    div Добро пожаловать. <br>Приступить к <router-link to="/game">игре</router-link>
</template>

<script>

export default {
  name: 'Home',
};
</script>
