<template>
  <div class="crud-card">
    <h2 class="crud-subtitle">⚠️ Impresión de Otras Etiquetas</h2>

    <div class="form-grid">
      <div class="form-field">
        <label class="crud-label">Tipo de Etiqueta</label>
        <select v-model="tipoEtiqueta" class="crud-input" :disabled="isPrinting">
          <option value="fragil">Frágil</option>
          <option value="hacia_arriba">Hacia Arriba</option>
        </select>
      </div>

      <div class="form-field">
        <label class="crud-label">Cantidad</label>
        <input type="number" v-model="cantidad" min="1" class="crud-input" :disabled="isPrinting" />
      </div>
    </div>

    <div class="preview-container">
      <h3 class="preview-title">Vista Previa (Simulada ZPL)</h3>
      <div v-if="tipoEtiqueta === 'fragil'" class="preview-box">
        <span class="symbol">🍸</span> 
        <p>FRÁGIL</p>
      </div>
      <div v-else-if="tipoEtiqueta === 'hacia_arriba'" class="preview-box">
        <span class="symbol">⬆️⬆️</span>
        <p>ESTE LADO ARRIBA</p>
      </div>
    </div>

    <div class="form-actions">
      <button @click="imprimirEtiquetas" class="crud-button" :disabled="isPrinting">
        <span v-if="isPrinting">🖨️ Enviando...</span>
        <span v-else>🖨️ Imprimir</span>
      </button>
    </div>
  </div>
</template>

<script>
// URL de la API Central (donde generas el ZPL)
const CENTRAL_API_BASE_URL = "https://backend-upper.onrender.com"; // Tu servidor FastAPI
// URL del Micro-servicio Local (donde envías el ZPL a la impresora)
const LOCAL_PRINT_API_URL = "http://127.0.0.1:8001/print"; 

export default {
    name: "comp_otras_etiquetas",
    data() {
      return {
          tipoEtiqueta: "fragil",
          cantidad: 1,
          isPrinting: false, // Controlar el estado de carga
      };
    },
    methods: {
      async imprimirEtiquetas() {
          if (this.isPrinting) return; 
          
          if (!this.cantidad || this.cantidad < 1) {
            alert("Ingresa una cantidad válida (mínimo 1)");
            return;
          }

          this.isPrinting = true;
          let zplCode = "";
          let printSuccess = false;

          // --- PASO A: PRIMERA LLAMADA (API Central) - Obtener ZPL ---
          try {
            const centralApiUrl = `${CENTRAL_API_BASE_URL}/imprimir/generate_other_label`;
            const payload = {
              tipo_etiqueta: this.tipoEtiqueta,
              cantidad: parseInt(this.cantidad, 10),
            };

            console.log(`Paso A: Solicitando ZPL a: ${centralApiUrl}`, payload);

            const response = await fetch(centralApiUrl, {
              method: 'POST',
              headers: {
                  'Content-Type': 'application/json',
              },
              body: JSON.stringify(payload),
            });

            const data = await response.json();

            if (!response.ok || data.error) {
              throw new Error(`El servidor reportó un error: ${data.error || response.statusText}`);
            }
          
            zplCode = data.zpl_code;
            if (!zplCode || zplCode.length === 0) throw new Error("La API Central devolvió un código ZPL vacío.");
            
            console.log("Paso A completado: ZPL generado (longitud: " + zplCode.length + ")");
            
          } catch (error) {
            console.error('❌ Error al generar ZPL:', error);
            alert(`❌ Error al generar ZPL. ¿Está la API Central (${CENTRAL_API_BASE_URL}) funcionando?\nDetalle: ${error.message}`); 
            this.isPrinting = false;
            return;
          }

          // --- PASO B: SEGUNDA LLAMADA (Micro-servicio Local) - Enviar ZPL a USB ---
          try {
            console.log(`Paso B: Enviando ZPL a servicio local: ${LOCAL_PRINT_API_URL}`);
            
            const printResponse = await fetch(LOCAL_PRINT_API_URL, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({ zpl_code: zplCode }),
            });
            
            const printData = await printResponse.json();
            
            if (!printResponse.ok || printData.error) {
                 throw new Error(`El servicio local reportó un error: ${printData.error || printResponse.statusText}`);
            }

            console.log("✅ Impresión USB exitosa:", printData);
            printSuccess = true;

          } catch (error) {
            console.error("❌ Error al imprimir en servicio local:", error);
            alert(`❌ Error de impresión local. Asegúrate de que el **Micro-servicio Local** (${LOCAL_PRINT_API_URL}) esté corriendo en la máquina con la impresora USB. \nDetalle: ${error.message}`);
          } 
          
          finally {
            this.isPrinting = false;
            if (printSuccess) {
                alert(`✅ Éxito! Se enviaron ${this.cantidad} etiquetas de "${this.tipoEtiqueta}" a la impresora local.`);
            }
          }
      },
    },
};
</script>
