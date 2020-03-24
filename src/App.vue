<template>
  <div id="app">
    <div id="nav">
      <router-link to="/">Play</router-link> |
      <router-link to="/about">About</router-link>
    </div>
    <router-view/>
    <footer>made by <a href="//rabol.co">Frederik Rab&oslash;l</a></footer>
  </div>
</template>

<style lang="scss">
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  height: 100%;
}
#nav {
  padding: 30px;
  a {
    font-weight: bold;
    color: #2c3e50;
    &.router-link-exact-active {
      color: #42b983;
    }
  }
}
html, body { padding: 0; margin: 0; width: 100%; height: 100%; }
</style>
<style lang="scss" scoped>
footer {
  position: fixed;
  display: block;
  bottom:0;
  left: 0;
  right: 0;
  background-color: #2c3e50;
  padding: 0.5rem;
  color: turquoise;
  a {
    color: inherit;
    font-weight: bold;
    text-decoration: none;
    &:hover {
      text-decoration: underline;
    }
  }
}
</style>
