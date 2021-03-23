<template lang="pug">
#app
  h1 NodeJS VM Browser Implementation tester
  p.text-wrapper.text
    .
      A small project, built as part of my engineering practice, to outline potential shortcomings of web-based polyfills for NodeJS's vm.
      The context here is, we were developing a solution for running machine-learning algorithms from a web-based no-code editor.
      But for optimal usability, this requires some sort of real-time feedback, which I figured is best pushed to the user's browser.
      (Since the server we are working on will likely die, if you hit it with too many clients at once)
      There's just a slight problem, the backend's code is purely written in Python, and well, web-browsers don't really like that language.
      But directly transpiling the Python code to JavaScript and sending that to the clients doesn't seem like a great idea either...
      Which is why I developed a solution which first transpiles the Python to LUA, and then, using fengari runs that in the browser.
      While that's not ultra-secure either, it should help avoid some mistakes :)
      During some research I came accross NodeJS's VM, which provides a separate context for executing JS-Code.
    br
    b Note: THIS IS NOT INTENDED AS A SECURITY MECHANISM
    .
      &nbsp;it's fairly easy to break, as shown by 
    a(href="https://gist.github.com/jcreedcmu") jcreedcmu
    .
      &nbsp;
    a(href="https://gist.github.com/jcreedcmu/4f6e6d4a649405a9c86bb076905696af") here
    .
      &nbsp;this can fairly easily be circumvented.
      But anyways, there are browser-implementations of it (They all use an iframe in some sort of way), so I thought, let's test them, maybe they could be useful...
    br
    .
      The thought here was - browsers implement their fair share of security mechanisms, maybe it works better here?
      But as you can see below (Live Demo, by the way), they're also not that great...
    br
    .
      Do note, these exploits are fairly simple, so if something shows up as "secure", that doesn't mean you can trust it :)
  .content-wrapper
    .demo-wrapper
      .demo.card(v-for="lut of results")
        h2 {{ lut.name }}
        .header
          .name Test name
          .result result
        .test(v-for="test of lut.tests")
          .name {{ test.name }}
          .result(v-if="test.result") ✅
            .description.card 
              b leak found
              br
              .
                {{ test.description }}
          .result(v-if="!test.result") ❎
            .description.card
              b no leak found
              br
              .
                {{ test.description }}
          
  #secret.hidden cake
</template>

<script>

import VMBrowserify from 'vm'
import VMBrowser from '@/lib/vmbrowserModule'

const wrapFunction = func => `(${func.toString()})()`

const checkCookieLeak = () => {
  const cookies = Object.fromEntries(document.cookie.split('; ').map(v=>v.split('=').map(decodeURIComponent)));
  return cookies['secret'] === 'cake';
}

const checkLocalStorageLeak = () => {
  return window.localStorage.getItem('secret') === 'cake';
}

const checkHTMLLeak = () => {
  const htmlSecret = document.getElementById('secret');
  return htmlSecret?.innerText === 'cake';
}


const allTests = [
  {
    name: 'cookie',
    description: 'Checks if a cookie set via document.cookie can be accessed from inside the VM',
    f: checkCookieLeak
  }, {
    name: 'localStorage',
    description: 'Checks if data written to localStorage is accessible from inside the VM',
    f: checkLocalStorageLeak
  }, {
    name: 'html',
    description: 'Checks if the VM has access to the HTMl-Document',
    f: checkHTMLLeak
  }]

export default {
  name: 'App',

  data() {
    return {
      results: [],
    }
  },

  mounted(){
    this.initializeTests();
    this.results.push({name: 'vm-browserify', tests: this.testVMBrowserify()});
    this.results.push({name: 'vm-browser', tests: this.testVMBrowser()});
    console.log(this.browserifyResults)
  },

  methods: {
    initializeTests(){
      // Set a cookie
      document.cookie = 'secret=cake; path=/'
      // set some local storage
      window.localStorage.setItem('secret', 'cake');
    },

    runTest(test, runner) {
      const { name, description, f } = test;
      const result = runner(f);
      return { name, description, result };
    },

    testVMBrowserify() {
      const testRunner = (f) => VMBrowserify.runInNewContext(wrapFunction(f), {});
      return allTests.map(test => this.runTest(test, testRunner));
    },

    testVMBrowser() {
      const sandbox = {}
      const context = VMBrowser.createContext(sandbox);
      const testRunner = (f) => {
        const script = new VMBrowser.Script(wrapFunction(f));
        return script.runInContext(context);
      }
      return allTests.map(test => this.runTest(test, testRunner));
    }
  }
}
</script>

<style lang="scss">
@import url('https://fonts.googleapis.com/css2?family=Montserrat:ital,wght@0,100;0,200;0,300;0,400;0,500;0,600;0,700;0,900;1,300&display=swap');

body {
  margin: 0;
  box-sizing: border-box;
}

#app {
  font-family: 'Montserrat', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;

  background: #fefefe;
  color: #2c3e50;

  display: flex;
  flex-direction: column;
  width: 100vw;
  height: 100vh;
}

h1 {
  font-weight: 800;
  font-size: 2.4rem;
}

h2 {
  font-weight: 800;
}

.text-wrapper {
  width: 90%;
  max-width: 700px;
  margin: 0 auto;
  text-align: justify;
}

.text {
  font-size: 1.15rem;
  
  &::after {
    content: " ";
  }

  &.bold {
    font-weight: 600;
  }

  a {
    display: inline-block;
  }
}

.card {
  background: #F9F9F9;
  margin: 0 4rem;

  border-radius: 10px;
  box-shadow: 0px 0px 32px 8px rgba($color: #000000, $alpha: 0.1);
}

.content-wrapper {
  flex: 1 1 0;
  display: flex;
  flex-direction: column;
}

.demo-wrapper {
  flex: 0 1 0;
  display: flex;
  flex-direction: row;
  justify-content: center;
  margin: auto;
  width: 100%;

  padding: 2rem 0;

  .demo {
    position: relative;
    width: 40%;
    max-width: 400px;
    background: #F9F9F9;
    margin: 0 4rem;

    border-radius: 10px;
    box-shadow: 0px 0px 32px 8px rgba($color: #000000, $alpha: 0.1);

    h2 {
      margin: 2rem 0;
    }

    .header,
    .test {
      display: grid;
      grid-template-columns: 5fr 5fr;
      
      .name {
        font-weight: 800;
      }
    }

    .test {
      margin: 2rem 0;
    }

    .result {
      position: relative;
      cursor: default;
    }


    .description {
      z-index: 10;
      opacity: 0;
      display: block;
      position: absolute;
      top: -22px;
      right: 30%;
      margin: 0;
      transform: translateX(100%);
      width: 200px;

      background: #F9F9F9;
      padding: 1rem;

      transition: opacity 100ms ease-in-out;

      &::after {
       content: " ";
       display: block;
       width: 32px;
       height: 32px;
       position: absolute;
       left: -8px;
       top: 16px;

       z-index: -1;

       transform: rotateZ(45deg);

       background: #F9F9F9;
     }
    }

    .result:hover > .description {
      opacity: 1;
    }



    .header {
      border-bottom: 1px solid rgba($color: #000000, $alpha: 0.1);
      margin: 1rem 0 2rem 0;
    }
  }
}


.hidden {
  display: none;
}

</style>
