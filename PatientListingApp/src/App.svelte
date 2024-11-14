<script lang="ts">
  import { Router, Link, Route } from "svelte-routing";
  import { navigate } from "svelte-routing";
  import PatientLista from "./PatientLista.svelte";
  import PatientForm from "./PatientCreate.svelte";
  import PatientModify from "./PatientModify.svelte";
  import PatientObservations from "./PatientObservations.svelte";
  import PatientAppointments from "./PatientAppointments.svelte";
  import PatientQuestionnaires from "./PatientQuestionnaires.svelte"; // Import the new component

  export let url = "";

  function handleLogoClick() {
      navigate("/");
  }
</script>

<Router {url}>
  <header>
      <div class="logo-container">
          <button class="logo-button" on:click={handleLogoClick}>
              <img src="/src/PatientLogo.png" alt="Patient Listing App Logo" class="logo">
          </button>
      </div>
      <nav>
          <Link to="/">Patient List</Link>
          <Link to="/create">Create Patient</Link>
      </nav>
  </header>

  <main>
      <div class="content-container">
          <Route path="/" component={PatientLista} />
          <Route path="/create" component={PatientForm} />
          <Route path="/modify/:id" let:params>
              <PatientModify id={params.id} />
          </Route>
          <Route path="/observations/:id" component={PatientObservations} />
          <Route path="/appointments/:id" component={PatientAppointments} />
          <Route path="/questionnaires/:id" component={PatientQuestionnaires} /> <!-- New route for Questionnaires -->
      </div>
  </main>
</Router>

<style>
    
  header {
      display: flex;
      flex-direction: column;
      align-items: center;
  }
  .logo-container {
      text-align: center;
      margin-bottom: 20px;
      background-color: #ffffff;
      width: 100%;
  }
  .logo-button {
      background: none;
      border: none;
      cursor: pointer;
      padding: 0;
  }
  .logo {
      max-width: 130px;
      height: auto;
  }
  nav {
      margin-bottom: 10px;
      background-color: hsla(204, 6%, 33%, 0.438);
      width: 100%;
  }
  nav :global(a) {
      margin-right: 25px;
      text-decoration: none;
      color: #ffffff;
      font-size: 20px;
  }
  nav :global(a:hover) {
      text-decoration: underline;
  }
  main {
      padding: 20px;
  }
</style>