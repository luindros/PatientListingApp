<script lang="ts">
  import { onMount } from 'svelte';
  import axios from "axios";
  import { navigate } from "svelte-routing";

  const hapiFhir = "https://hapi.fhir.org/baseR4";
  let patients: any[] = [];
  let searchTerm = "";
  let page = 0;

  function isPhoneNumber(input: string): boolean {
      const phoneRegex = /^[\+]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{3}[-\s\.]?[0-9]{4,6}$/;
      return phoneRegex.test(input.replace(/\s/g, ''));
  }

  async function fetchPatients(search = "", newPage = 0) {
      page = newPage;
      try {
          const params: any = {
              _sort: "-_lastUpdated",
              _count: 20,
              _offset: newPage * 20
          };

          if (search) {
              if (isPhoneNumber(search)) {
                  params.telecom = `phone|${search}`;
              } else {
                  params.name = search;
              }
          }

          const response = await axios.get(`${hapiFhir}/Patient`, { params });
          patients = response.data.entry?.map((entry: any) => entry.resource) || [];
      } catch (error) {
          console.error("Error fetching patients:", error);
      }
  }

  function handleSearch() {
      fetchPatients(searchTerm, page);
  }

  function handleKeyPress(event: KeyboardEvent) {
      if (event.key === 'Enter') {
          handleSearch();
      }
  }

  function handlePatientClick(patientId: string) {
      navigate(`/modify/${patientId}`);
  }

  function refreshPage() {
      window.location.reload();
  }

  function handleObservationsClick(event: Event, patientId: string) {
      event.stopPropagation();
      navigate(`/observations/${patientId}`);
  }

  function handleAppointmentsClick(event: Event, patientId: string) {
      event.stopPropagation();
      navigate(`/appointments/${patientId}`);
  }

  function handleQuestionnairesClick(event: Event, patientId: string) {
      event.stopPropagation();
      navigate(`/questionnaires/${patientId}`);
  }

  onMount(() => fetchPatients("", 0));
</script>

<main>
  <div>
      <div class="search-container">
          <input 
              bind:value={searchTerm} 
              placeholder="Search by name or phone number"
              on:keypress={handleKeyPress}
          />
          <button on:click={handleSearch}>Search</button>
          <button class="refresh-button" on:click={refreshPage} title="Refresh page">↻</button>
      </div>
      <table>
          <thead>
              <tr>
                  <th>ID</th>
                  <th>Name</th>
                  <th>Gender</th>
                  <th>Birth Date</th>
                  <th>Last Updated</th>
                  <th>Phone Number</th>
                  <th>Observations</th> 
                  <th>Appointments</th> 
                  <th>Questionnaires</th> <!-- New header for Questionnaires -->
              </tr>
          </thead>
          <tbody>
              {#each patients as patient}
                  <tr on:click={() => handlePatientClick(patient.id)}>
                      <td>{patient.id}</td>
                      <td>{patient.name?.[0]?.family}, {patient.name?.[0]?.given?.join(' ') || ''}</td>
                      <td>{patient.gender || 'Unknown'}</td>
                      <td>{patient.birthDate || 'Unknown'}</td>
                      <td>{new Date(patient.meta?.lastUpdated).toLocaleString()}</td>
                      <td>{patient.telecom?.[0]?.value || 'Unknown'}</td>
                      <td>
                          <button class="observations-button" on:click={(e) => handleObservationsClick(e, patient.id)}>Observations</button>
                      </td>
                      <td>
                          <button class="observations-button" on:click={(e) => handleAppointmentsClick(e, patient.id)}>Appointments</button>
                      </td>
                      <td>
                          <button class="observations-button" on:click={(e) => handleQuestionnairesClick(e, patient.id)}>Questionnaires</button> <!-- New button for Questionnaires -->
                      </td>
                  </tr>
              {/each}
          </tbody>
      </table>
      <div class="pagination-container">
          <button on:click={() => fetchPatients("", page-1)}>Previous</button>
          <span>Page {page + 1}</span>
          <button on:click={() => fetchPatients('', page + 1)}>Next</button>
      </div>
  </div>
</main>

<style>
  main {
      font-family: Arial, sans-serif;
      max-width: 1200px;
      margin: 0 auto;
      padding: 20px;
  }

  .search-container {
      display: flex;
      margin-bottom: 20px;
  }

  input {
      flex-grow: 1;
      padding: 10px;
      font-size: 16px;
      border: 1px solid #ddd;
      border-radius: 4px 0 0 4px;
  }

  button {
      padding: 10px 20px;
      font-size: 16px;
      background-color: #48a2ad;
      color: white;
      border: none;
      border-radius: 0 4px 4px 0;
      cursor: pointer;
      transition: background-color 0.3s ease;
  }

  button:hover {
      background-color:  #82edfd;
  }

  .refresh-button {
      padding: 10px 15px;
      font-size: 20px;
      background-color: #4CAF50;
      border-radius: 4px;
      margin-left: 10px;
  }

  .refresh-button:hover {
      background-color: #45a049;
  }

  table {
      width: 100%;
      border-collapse: separate;
      border-spacing: 0;
      margin-top: 20px;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
      border-radius: 8px;
      overflow: hidden;
  }

  th, td {
      padding: 12px;
      text-align: left;
      border-bottom: 1px solid #e0e0e0;
  }

  th {
      background-color: #00adc0;
      color: white;
      font-weight: bold;
      text-transform: uppercase;
      letter-spacing: 0.5px;
  }

  tr:last-child td {
      border-bottom: none;
  }

  tbody tr:hover {
      background-color: #a8b4b6;
      cursor: pointer;
      transition: background-color 0.3s ease;
  }

  .pagination-container {
      margin-top: 20px;
  }
</style>