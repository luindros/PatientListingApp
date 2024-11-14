<script lang="ts">
    import { onMount } from 'svelte';
    import { navigate } from "svelte-routing";
    import axios from "axios";
  
    export let id: string;
    const hapiFhir = "https://hapi.fhir.org/baseR4";
    let appointments: any[] = [];
    let patient: any = null;
    let loading = true;
    let error = false;
    let showForm = false;
    let newAppointment = {
      start: "",
      end: "",
      status: "booked",
      description: "",
      type: ""
    };
  
    // List of common appointment types
    const appointmentTypes = [
      { code: "408443003", display: "General medical practice" },
      { code: "394701000", display: "Cardiology" },
      { code: "394584008", display: "Dermatology" },
      { code: "394582007", display: "Neurology" },
      { code: "394579002", display: "Dentistry" },
      { code: "394585009", display: "Orthopedics" },
      { code: "394589003", display: "Psychiatry" },
      { code: "394591006", display: "Endocrinology" }
    ];
  
    onMount(async () => {
      await fetchAppointments();
    });
  
    async function fetchAppointments() {
      loading = true;
      error = false;
      try {
        const [patientResponse, appointmentsResponse] = await Promise.all([
          axios.get(`${hapiFhir}/Patient/${id}`),
          axios.get(`${hapiFhir}/Appointment`, {
            params: {
              patient: id,
              _sort: '-date',
              _count: 100
            }
          })
        ]);
  
        patient = patientResponse.data;
        appointments = appointmentsResponse.data.entry?.map((entry: any) => entry.resource) || [];
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
  
    async function createAppointment() {
      try {
        const appointmentData = {
          resourceType: "Appointment",
          status: newAppointment.status,
          start: new Date(newAppointment.start).toISOString(),
          end: new Date(newAppointment.end).toISOString(),
          serviceType: [
            {
              coding: [
                {
                  system: "http://snomed.info/sct",
                  code: newAppointment.type,
                  display: appointmentTypes.find(t => t.code === newAppointment.type)?.display
                }
              ]
            }
          ],
          participant: [
            {
              actor: {
                reference: `Patient/${id}`
              },
              status: "accepted"
            }
          ],
          description: newAppointment.description
        };
  
        await axios.post(`${hapiFhir}/Appointment`, appointmentData);
        showForm = false;
        newAppointment = { start: "", end: "", status: "booked", description: "", type: "" };
        await fetchAppointments();
      } catch (err) {
        console.error("Error creating appointment:", err);
        alert("Failed to create appointment. Please try again.");
      }
    }
  
    function refreshAppointments() {
      showForm = false; // Hide the form when refreshing
      fetchAppointments();
    }
  
    async function deleteAppointment(appointmentId: string) {
      try {
        await axios.delete(`${hapiFhir}/Appointment/${appointmentId}`);
        await fetchAppointments(); // Refresh the appointments list
      } catch (err) {
        console.error("Error deleting appointment:", err);
        alert("Failed to delete appointment. Please try again.");
      }
    }
  </script>
  
  <main>
    <div class="container">
      <h1 class="title">Patient Appointments</h1>
      {#if loading}
        <p>Loading appointments...</p>
      {:else if error}
        <p class="error">Error loading appointments. Please try again.</p>
      {:else}
        <h2 class="patientName">{patient.name[0].given[0]} {patient.name[0].family} ({patient.id})</h2>
        
        <div class="button-container">
          <button on:click={toggleForm}>
            {showForm ? 'Hide Form' : 'Create New Appointment'}
          </button>
          <button class="refresh-button" on:click={refreshAppointments}>
            &#x21bb;
          </button>
        </div>
        
        {#if showForm}
          <form on:submit|preventDefault={createAppointment} class="appointment-form">
            <input type="datetime-local" bind:value={newAppointment.start} required>
            <input type="datetime-local" bind:value={newAppointment.end} required>
            <select bind:value={newAppointment.status}>
              <option value="booked">Booked</option>
              <option value="pending">Pending</option>
              <option value="arrived">Arrived</option>
            </select>
            <select bind:value={newAppointment.type} required>
              <option value="">Select appointment type</option>
              {#each appointmentTypes as type}
                <option value={type.code}>{type.display}</option>
              {/each}
            </select>
            <input type="text" bind:value={newAppointment.description} placeholder="Description">
            <button type="submit">Create Appointment</button>
          </form>
        {/if}
        
        {#if appointments.length === 0}
          <p>No appointments found for this patient.</p>
        {:else}
          <table>
            <thead>
              <tr>
                <th>Date</th>
                <th>Status</th>
                <th>Type</th>
                <th>Description</th>
                <th>Action</th>
              </tr>
            </thead>
            <tbody>
              {#each appointments as appointment}
                <tr>
                  <td>{formatDate(appointment.start)}</td>
                  <td>{appointment.status}</td>
                  <td>{appointment.serviceType?.[0]?.coding?.[0]?.display || 'N/A'}</td>
                  <td>{appointment.description || 'N/A'}</td>
                  <td>
                    <button class="delete-button" on:click={() => deleteAppointment(appointment.id)}>
                      Delete
                    </button>
                  </td>
                </tr>
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
      color: rgb(0, 0, 0);
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
      background-color: #0b4b57;
      transition: background-color 0.3s ease;
    }
  
    .error {
      color: red;
      font-weight: bold;
    }
  
    .appointment-form {
      display: flex;
      flex-direction: column;
      gap: 10px;
      margin-top: 20px;
      padding: 20px;
      background-color: #f0f0f0;
      border-radius: 8px;
    }
  
    .appointment-form input,
    .appointment-form select,
    .appointment-form button {
      padding: 10px;
      font-size: 16px;
    }
  
    .appointment-form button {
      background-color: #48a2ad;
      color: white;
      border: none;
      cursor: pointer;
      transition: background-color 0.3s ease;
    }
  
    .appointment-form button:hover {
      background-color: #82edfd;
    }
  </style>