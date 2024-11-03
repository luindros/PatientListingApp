<script lang="ts">
  import { onMount } from 'svelte';
  import { navigate } from "svelte-routing";
  import axios from "axios";

  export let id: string;
  const hapiFhir = "https://hapi.fhir.org/baseR4";
  let observations: any[] = [];
  let patient: any = null;
  let loading = true;
  let error = false;
  let expandedRows: Set<number> = new Set();
  let showForm = false;
  let newObservation = {
    code: "",
    effectiveDateTime: "",
    components: [{ code: "", value: "", unit: "" }]
  };

  // List of common observation types
  const observationTypes = [
    { code: "8867-4", display: "Heart rate" },
    { code: "8480-6", display: "Blood pressure" },
    { code: "8310-5", display: "Body temperature" },
    { code: "8302-2", display: "Body height" },
    { code: "29463-7", display: "Body weight" },
    { code: "8462-4", display: "Diastolic blood pressure" },
    { code: "8480-6", display: "Systolic blood pressure" },
    { code: "59408-5", display: "Oxygen saturation" }
  ];

  onMount(async () => {
    await fetchObservations();
  });

  async function fetchObservations() {
    loading = true;
    error = false;
    try {
      const [patientResponse, observationsResponse] = await Promise.all([
        axios.get(`${hapiFhir}/Patient/${id}`),
        axios.get(`${hapiFhir}/Observation`, {
          params: {
            patient: id,
            _sort: '-date',
            _count: 100
          }
        })
      ]);

      patient = patientResponse.data;
      observations = observationsResponse.data.entry?.map((entry: any) => entry.resource) || [];
    } catch (err) {
      console.error("Error fetching data:", err);
      error = true;
    } finally {
      loading = false;
    }
  }

  function formatDate(dateString: string): string {
    return new Date(dateString).toLocaleString();
  }

  function goBack() {
    navigate(`/`);
  }

  function toggleForm() {
    showForm = !showForm;
  }

  function toggleRow(index: number) {
    if (expandedRows.has(index)) {
      expandedRows.delete(index);
    } else {
      expandedRows.add(index);
    }
    expandedRows = expandedRows;
  }

  function getObservationValue(observation: any): string {
    if (observation.valueQuantity) {
      return `${observation.valueQuantity.value} ${observation.valueQuantity.unit || ''}`;
    } else if (observation.valueCodeableConcept) {
      return observation.valueCodeableConcept.text || 'N/A';
    } else if (observation.valueString) {
      return observation.valueString;
    } else {
      return 'N/A';
    }
  }

  function addComponent() {
    newObservation.components = [...newObservation.components, { code: "", value: "", unit: "" }];
  }

  function removeComponent(index: number) {
    newObservation.components = newObservation.components.filter((_, i) => i !== index);
  }

  async function createObservation() {
    try {
      const observationData: any = {
        resourceType: "Observation",
        status: "final",
        code: {
          coding: [
            {
              system: "http://loinc.org",
              code: newObservation.code,
              display: observationTypes.find(t => t.code === newObservation.code)?.display
            }
          ]
        },
        subject: {
          reference: `Patient/${id}`
        },
        effectiveDateTime: new Date(newObservation.effectiveDateTime).toISOString(),
      };

      if (newObservation.components.length === 1) {
        observationData.valueQuantity = {
          value: parseFloat(newObservation.components[0].value),
          unit: newObservation.components[0].unit,
          system: "http://unitsofmeasure.org"
        };
      } else {
        observationData.component = newObservation.components.map(comp => ({
          code: {
            coding: [
              {
                system: "http://loinc.org",
                code: comp.code,
                display: observationTypes.find(t => t.code === comp.code)?.display
              }
            ]
          },
          valueQuantity: {
            value: parseFloat(comp.value),
            unit: comp.unit,
            system: "http://unitsofmeasure.org"
          }
        }));
      }

      await axios.post(`${hapiFhir}/Observation`, observationData);
      showForm = false;
      newObservation = {
        code: "",
        effectiveDateTime: "",
        components: [{ code: "", value: "", unit: "" }]
      };
      await fetchObservations();
    } catch (err) {
      console.error("Error creating observation:", err);
      alert("Failed to create observation. Please try again.");
    }
  }

  function refreshObservations() {
    showForm = false;
    fetchObservations();
  }

  async function deleteObservation(observationId: string) {
    try {
      await axios.delete(`${hapiFhir}/Observation/${observationId}`);
      await fetchObservations();
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
      <h2 class="patientName">{patient.name[0].given[0]} {patient.name[0].family} ({patient.id})</h2>
      
      <div class="button-container">
        <button on:click={toggleForm}>
          {showForm ? 'Hide Form' : 'Create New Observation'}
        </button>
        <button class="refresh-button" on:click={refreshObservations}>
          &#x21bb;
        </button>
      </div>
      
      {#if showForm}
        <form on:submit|preventDefault={createObservation} class="observation-form">
          <select bind:value={newObservation.code} required>
            <option value="">Select observation type</option>
            {#each observationTypes as type}
              <option value={type.code}>{type.display}</option>
            {/each}
          </select>
          <input type="datetime-local" bind:value={newObservation.effectiveDateTime} required>
          
          {#each newObservation.components as component, index}
            <div class="component">
              <select bind:value={component.code} required>
                <option value="">Select component type</option>
                {#each observationTypes as type}
                  <option value={type.code}>{type.display}</option>
                {/each}
              </select>
              <input type="number" bind:value={component.value} placeholder="Value" required>
              <input type="text" bind:value={component.unit} placeholder="Unit" required>
              {#if index > 0}
                <button type="button" on:click={() => removeComponent(index)}>Remove</button>
              {/if}
            </div>
          {/each}
          
          <button type="button" on:click={addComponent}>Add Component</button>
          <button type="submit">Create Observation</button>
        </form>
      {/if}
      
      {#if observations.length === 0}
        <p>No observations found for this patient.</p>
      {:else}
        <table>
          <thead>
            <tr>
              <th>Date</th>
              <th>Type</th>
              <th>Value</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {#each observations as observation, index}
              <tr>
                <td>{formatDate(observation.effectiveDateTime)}</td>
                <td>{observation.code.coding[0].display}</td>
                <td>{getObservationValue(observation)}</td>
                <td>
                  <button class="delete-button" on:click={() => deleteObservation(observation.id)}>
                    Delete
                  </button>
                  {#if observation.component}
                    <button on:click={() => toggleRow(index)}>
                      {expandedRows.has(index) ? 'Hide Details' : 'Show Details'}
                    </button>
                  {/if}
                </td>
              </tr>
              {#if observation.component && expandedRows.has(index)}
                <tr class="expanded-row">
                  <td colspan="4">
                    <table class="inner-table">
                      <thead>
                        <tr>
                          <th>Component</th>
                          <th>Value</th>
                        </tr>
                      </thead>
                      <tbody>
                        {#each observation.component as component}
                          <tr>
                            <td>{component.code?.coding?.[0]?.display || 'Unknown'}</td>
                            <td>{getObservationValue(component)}</td>
                          </tr>
                        {/each}
                      </tbody>
                    </table>
                  </td>
                </tr>
              {/if}
            {/each}
          </tbody>
        </table>
      {/if}
      <div class="button-container">
        <button on:click={goBack}>Back</button>
      </div>
    {/if}
  </div>
</main>

<style>
  .title {
    text-align: center;
    color: rgb(35, 166, 184);
  }
  .patientName {
    text-align: center;
    color: rgb(104, 237, 255);
  }
  main {
    font-family: Arial, sans-serif;
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
  }

  .container {
    width: 100%;
  }

  h1, h2 {
    color: #333;
    text-align: center;
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

  .refresh-button {
    padding: 10px 15px;
    font-size: 20px;
    background-color: #4CAF50;
    border-radius: 4px;
  }

  .refresh-button:hover {
    background-color: #45a049;
  }

  .delete-button {
    background-color: #f44336;
    padding: 5px 10px;
    font-size: 14px;
  }

  .delete-button:hover {
    background-color: #d32f2f;
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

  .expanded-row {
    background-color: #243b44;
  }

  .inner-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 10px;
  }

  .inner-table th, .inner-table td {
    border: 1px solid #ddd;
    padding: 8px;
  }

  .inner-table th {
    background-color: #e9e9e9;
    color: #333;
  }

  .error {
    color: red;
    font-weight: bold;
  }

  .observation-form {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-top: 20px;
    padding: 20px;
    background-color: #f0f0f0;
    border-radius: 8px;
  }

  .observation-form input,
  .observation-form select,
  .observation-form button {
    padding: 10px;
    font-size: 16px;
  }

  .observation-form button {
    background-color: #48a2ad;
    color: white;
    border: none;
    cursor: pointer;
    transition: background-color 0.3s ease;
  }

  .observation-form button:hover {
    background-color: #82edfd;
  }

  .component {
    display: flex;
    gap: 10px;
    align-items: center;
  }

  .component button {
    padding: 5px 10px;
    font-size: 14px;
  }
</style>