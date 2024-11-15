<script lang="ts">
    import { onMount } from 'svelte';
    import { navigate } from "svelte-routing";
    import axios from "axios";

    export let id: string; // Patient ID
    const hapiFhir = "https://hapi.fhir.org/baseR4";
    let observations: any[] = [];
    let patient: any = null; // To store patient details
    let loading = true;
    let error = false;

    onMount(async () => {
        await fetchPatientDetails(); // Fetch patient details
        await fetchObservationsWithEncounters(); // Fetch observations with encounters
    });

    async function fetchPatientDetails() {
        try {
            const response = await axios.get(`${hapiFhir}/Patient/${id}`);
            patient = response.data; // Store patient details
        } catch (err) {
            console.error("Error fetching patient details:", err);
            error = true;
        }
    }

    async function fetchObservationsWithEncounters() {
        loading = true;
        error = false;
        try {
            const response = await axios.get(`${hapiFhir}/Observation`, {
                params: {
                    patient: id,
                    _include: "Observation:encounter", // Include encounter information
                    _sort: '-date',
                    _count: 100
                }
            });
            
            // Create a map of encounter identifiers for easy access
            const encounterMap = response.data.entry
                ?.filter(entry => entry.resource.resourceType === 'Encounter')
                .reduce((map, entry) => {
                    const encounterId = entry.resource.identifier?.[0]?.value; // Get the identifier value
                    if (encounterId) {
                        map[entry.resource.id] = encounterId; // Map encounter ID to its reference ID
                    }
                    return map;
                }, {});

            // Filter the response to extract only Observation resources
            observations = response.data.entry
                ?.filter(entry => entry.resource.resourceType === 'Observation') // Only keep Observations
                .map(entry => {
                    const observation = entry.resource;
                    const encounterReference = observation.encounter?.reference.split('/')[1]; // Get the encounter reference ID
                    return {
                        ...observation,
                        encounterId: encounterMap[encounterReference] || 'N/A' // Add the encounter ID from the map
                    };
                }) || [];
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
            const componentText = component.code?.coding?.[0]?.display || 'Unknown'; // Safe access
            return `${componentText}: ${componentValue}`;
        }).join(', ');
    }

    async function deleteObservation(observationId: string) {
        try {
            await axios.delete(`${hapiFhir}/Observation/${observationId}`);
            await fetchObservationsWithEncounters(); // Refresh the list after deletion
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
            {#if patient}
                <h2 class="patientName">{patient.name[0].given.join(' ')} {patient.name[0].family} ({patient.id})</h2>
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
                            <th>Encounter ID</th>
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {#each observations as observation}
                            <tr>
                                <td>{formatDate(observation.effectiveDateTime)}</td>
                                <td>{observation.code?.coding?.[0]?.display || 'N/A'}</td> <!-- Safe access -->
                                <td>
                                    {#if observation.component && observation.component.length > 0}
                                        {renderComponents(observation.component)} <!-- Display component values -->
                                    {:else}
                                        {getObservationValue(observation)} <!-- Display main observation value -->
                                    {/if}
                                </td>
                                <td>{observation.encounterId || 'N/A'}</td> <!-- Display Encounter ID -->
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

    .patientName {
        text-align: center;
        color: rgb(0, 0, 0);
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