<script lang="ts">
  import { onMount } from 'svelte';
  import { navigate } from "svelte-routing";
  import axios from "axios";

  export let id: string;
  const hapiFhir = "https://hapi.fhir.org/baseR4";
  let observations: any[] = [];
  let loading = true;
  let error = false;

  onMount(async () => {
      await fetchObservations();
  });

  async function fetchObservations() {
      loading = true;
      error = false;
      try {
          const response = await axios.get(`${hapiFhir}/Observation`, {
              params: {
                  patient: id,
                  _sort: '-date',
                  _count: 100
              }
          });
          observations = response.data.entry?.map((entry: any) => entry.resource) || [];
      } catch (err) {
          console.error("Error fetching observations:", err);
          error = true;
      } finally {
          loading = false;
      }
  }

  function formatDate(dateString: string): string {
      return new Date(dateString).toLocaleString();
  }

  function getObservationValue(observation: any): string {
      if (observation.valueQuantity) {
          return `${observation.valueQuantity.value} ${observation.valueQuantity.unit || ''}`;
      } else if (observation.valueCodeableConcept) {
          return observation.valueCodeableConcept.text || 'N/A';
      } else if (observation.valueString) {
          return observation.valueString;
      } else if (observation.valueBoolean !== undefined) {
          return observation.valueBoolean ? 'Yes' : 'No';
      } else if (observation.valueInteger !== undefined) {
          return observation.valueInteger.toString();
      } else if (observation.valueDecimal !== undefined) {
          return observation.valueDecimal.toString();
      } else if (observation.valueDate) {
          return new Date(observation.valueDate).toLocaleDateString();
      } else {
          return 'N/A';
      }
  }

  function renderComponents(components: any[]): string {
      return components.map(component => {
          const componentValue = getObservationValue(component);
          const componentText = component.code?.coding[0]?.display || 'Unknown';
          return `${componentText}: ${componentValue}`;
      }).join(', ');
  }

  async function deleteObservation(observationId: string) {
      try {
          await axios.delete(`${hapiFhir}/Observation/${observationId}`);
          await fetchObservations(); // Refresh the list after deletion
      } catch (err) {
          console.error("Error deleting observation:", err);
          alert("Failed to delete observation. Please try again.");
      }
  }
</script>

<main>
  <div class="container">
      <h1 class="title">Patient Observations</h1>
      {#if loading}
          <p>Loading observations...</p>
      {:else if error}
          <p class="error">Error loading observations. Please try again.</p>
      {:else}
          {#if observations.length === 0}
              <p>No observations found for this patient.</p>
          {:else}
              <table>
                  <thead>
                      <tr>
                          <th>Date</th>
                          <th>Type</th>
                          <th>Value</th> <!-- This column will now display various types -->
                          <th>Encounter ID</th>
                          <th>Actions</th>
                      </tr>
                  </thead>
                  <tbody>
                      {#each observations as observation}
                          <tr>
                              <td>{formatDate(observation.effectiveDateTime)}</td>
                              <td>{observation.code.coding[0]?.display || 'N/A'}</td>
                              <td>
                                  {#if observation.component && observation.component.length > 0}
                                      {renderComponents(observation.component)} <!-- Display component values -->
                                  {:else}
                                      {getObservationValue(observation)} <!-- Display main observation value -->
                                  {/if}
                              </td>
                              <td>{observation.encounter?.reference.split('/')[1] || 'N/A'}</td> <!-- Display Encounter ID -->
                              <td>
                                  <button class="delete-button" on:click={() => deleteObservation(observation.id)}>
                                      Delete
                                  </button>
                              </td>
                          </tr>
                      {/each}
                  </tbody>
              </table>
          {/if}
          <div class="button-container">
              <button on:click={() => navigate(-1)}>Back</button>
          </div>
      {/if}
  </div>
</main>

<style>
  .container {
      max-width: 1200px;
      margin: 0 auto;
      padding: 20px;
  }

  .title {
      text-align: center;
      color: rgb(35, 166, 184);
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
      background-color: #005f69;
      color: white;
      font-weight: bold;
      text-transform: uppercase;
      letter-spacing: 0.5px;
  }

  tr:last-child td {
      border-bottom: none;
  }

  tbody tr:hover {
      background-color: #0b4b57;
      transition: background-color 0.3s ease;
  }

  .error {
      color: red;
      font-weight: bold;
  }

  .button-container {
      display: flex;
      justify-content: center;
      margin-top: 20px;
  }

  button {
      padding: 10px 20px;
      font-size: 16px;
      background-color: #48a2ad;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      transition: background-color 0.3s ease;
      margin: 0 5px;
  }

  button:hover {
    background-color: #82edfd;
  }

  .delete-button {
      background-color: #f44336;
      padding: 5px 10px;
      font-size: 14px;
  }

  .delete-button:hover {
      background-color: #d32f2f;
  }
</style>