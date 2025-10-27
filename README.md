#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <limits.h>
#include <ctype.h>

#define MAX_NOMBRE 30
#define MAX_TIPO 20
#define TOTAL_SALAS 3
#define INIT_ASIENTOS 30
#define MAX_COLA 50

// ------------------------------
// Estructuras con memoria dinámica
// ------------------------------

typedef struct Asiento {
    int numero;
    char estado[10]; // "Libre" u "Ocupado"
    char nombreCliente[MAX_NOMBRE];
    char tipoCliente[MAX_TIPO];
    struct Asiento* siguiente;
} Asiento;

typedef struct NodoCliente {
    char nombre[MAX_NOMBRE];
    char tipo[MAX_TIPO];
    int prioridad;
    struct NodoCliente* siguiente;
} NodoCliente;

typedef struct {
    NodoCliente* frente;
    NodoCliente* final;
    int cantidad;
} ColaEspera;

typedef struct {
    int numeroSala;
    Asiento* asientos;
    int totalAsientos;
    int asientosLibres;
    ColaEspera cola;
} Sala;

// ------------------------------
// Prototipos de funciones
// ------------------------------
void inicializarSistema(Sala** salas);
void liberarSistema(Sala** salas);
void cicloSimulacion (Sala* sala);
void mostrarMenuPrincipal();
void mostrarBienvenida();
void gestionarReservas(Sala* sala);
void mostrarEstadoAsientos(Sala* sala);
void cancelarReserva(Sala* sala);
void gestionarColaEspera(Sala* sala);
void procesarFuncion(Sala* sala);
void guardarDatos(Sala* salas[]);
void cargarDatos(Sala** salas);
void pausa ();
void limpiarBuffer();

// Funciones de memoria dinámica
Asiento* crearAsiento(int numero);
void liberarAsientos(Asiento* asiento);
NodoCliente* crearNodoCliente(char nombre[], char tipo[], int prioridad);
void encolarCliente(ColaEspera* cola, char nombre[], char tipo[], int prioridad);
NodoCliente* desencolarCliente(ColaEspera* cola, int prioridad);
int obtenerPrioridad(const char* tipo);
bool validarTipoCliente(const char* tipo);
bool validarNombre(const char* nombre);
void simulacionAutomatica(Sala* salas[]);

// ------------------------------
// Implementación de funciones
// ------------------------------

void limpiarBuffer() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF);
}

void pausa() {
    printf("\nPresione Enter para continuar...");
    limpiarBuffer();
}

Asiento* crearAsiento(int numero) {
    Asiento* nuevo = (Asiento*)malloc(sizeof(Asiento));
    if(nuevo == NULL) {
        printf("Error al asignar memoria para asiento\n");
        exit(EXIT_FAILURE);
    }
    
    nuevo->numero = numero;
    strcpy(nuevo->estado, "Libre");
    strcpy(nuevo->nombreCliente, "");
    strcpy(nuevo->tipoCliente, "");
    nuevo->siguiente = NULL;
    
    return nuevo;
}

void liberarAsientos(Asiento* asiento) {
    while(asiento != NULL) {
        Asiento* temp = asiento;
        asiento = asiento->siguiente;
        free(temp);
    }
}

int obtenerPrioridad(const char* tipo) {
    if(strcasecmp(tipo, "Movilidad Reducida") == 0) return 1;
    if(strcasecmp(tipo, "VIP") == 0) return 2;
    return 3; // General
}
bool validarTipoCliente(const char* tipo) {
    return (strcasecmp(tipo, "General") == 0 || 
            strcasecmp(tipo, "VIP") == 0 || 
            strcasecmp(tipo, "Movilidad Reducida") == 0);
}
bool validarNombre(const char* nombre) {
    if(strlen(nombre) == 0) return false;
    while(*nombre) {
        if(!isalpha(*nombre) && *nombre != ' ') return false;
        nombre++;
    }
    return true;
}

NodoCliente* crearNodoCliente(char nombre[], char tipo[], int prioridad) {
    NodoCliente* nuevo = (NodoCliente*)malloc(sizeof(NodoCliente));
    if(nuevo == NULL) {
        printf("Error al asignar memoria para cliente\n");
        exit(EXIT_FAILURE);
    }
    
    strcpy(nuevo->nombre, nombre);
    strcpy(nuevo->tipo, tipo);
    nuevo->prioridad = prioridad;
    nuevo->siguiente = NULL;
    
    return nuevo;
}

void encolarCliente(ColaEspera* cola, char nombre[], char tipo[], int prioridad) {
    NodoCliente* nuevo = crearNodoCliente(nombre, tipo, prioridad);
    
    if(cola->frente == NULL) {
        cola->frente = nuevo;
    } else {
        cola->final->siguiente = nuevo;
    }
    
    cola->final = nuevo;
    cola->cantidad++;
}

NodoCliente* desencolarCliente(ColaEspera* cola, int prioridad) {
    if(cola->frente == NULL) return NULL;
    
    NodoCliente* actual = cola->frente;
    NodoCliente* anterior = NULL;
    NodoCliente* mejor = cola->frente;
    NodoCliente* anteriorMejor = NULL;
    
    // Buscar cliente con mayor prioridad (menor número)
    while(actual != NULL) {
        if(actual->prioridad < mejor->prioridad) {
            mejor = actual;
            anteriorMejor = anterior;
        }
        anterior = actual;
        actual = actual->siguiente;
    }
    
    // Reorganizar la cola
    if(anteriorMejor == NULL) {
        cola->frente = mejor->siguiente;
    } else {
        anteriorMejor->siguiente = mejor->siguiente;
    }
    
    if(mejor == cola->final) {
        cola->final = anteriorMejor;
    }
    
    cola->cantidad--;
    mejor->siguiente = NULL;
    return mejor;
}

void inicializarSistema(Sala** salas) {
    for(int i = 0; i < TOTAL_SALAS; i++) {
        salas[i] = (Sala*)malloc(sizeof(Sala));
        if(salas[i] == NULL) {
            printf("Error al asignar memoria para sala\n");
            exit(EXIT_FAILURE);
        }
        
        salas[i]->numeroSala = i + 1;
        salas[i]->totalAsientos = INIT_ASIENTOS;
        salas[i]->asientosLibres = INIT_ASIENTOS;
        
        // Crear lista de asientos
        Asiento* primero = crearAsiento(1);
        salas[i]->asientos = primero;
        
        Asiento* actual = primero;
        for(int j = 2; j <= INIT_ASIENTOS; j++) {
            actual->siguiente = crearAsiento(j);
            actual = actual->siguiente;
        }
        
        // Inicializar cola de espera
        salas[i]->cola.frente = NULL;
        salas[i]->cola.final = NULL;
        salas[i]->cola.cantidad = 0;
    }
}

void liberarSistema(Sala** salas) {
    for(int i = 0; i < TOTAL_SALAS; i++) {
        if(salas[i] != NULL) {
            // Liberar asientos
            liberarAsientos(salas[i]->asientos);
            
            // Liberar cola de espera
            NodoCliente* actual = salas[i]->cola.frente;
            while(actual != NULL) {
                NodoCliente* temp = actual;
                actual = actual->siguiente;
                free(temp);
            }
            
            free(salas[i]);
            salas[i] = NULL;
        }
    }
}

void mostrarBienvenida() {
    printf("\n========================================\n");
    printf(" BIENVENIDO AL SISTEMA DE GESTIÓN DE CINE\n");
    printf("========================================\n\n");
    printf("Cine Multipantalla - Gestión de Reservas\n");
    printf("Versión con Memoria Dinámica\n");
    pausa();
    system("clear || cls");
}

void mostrarMenuPrincipal() {
    printf("\n=== MENÚ PRINCIPAL ===\n");
    printf("1. Sala 1\n");
    printf("2. Sala 2\n");
    printf("3. Sala 3\n");
    printf("0. Salir\n");
}

void gestionarReservas(Sala* sala) {
    int opcion;
    do {
        printf("\n--- GESTIÓN DE RESERVAS ---\n");
        printf("1. Reservar asiento\n");
        printf("2. Cancelar reserva\n");
        printf("0. Volver\n");
        printf("Seleccione: ");
        
        if(scanf("%d", &opcion) != 1) {
            printf("Entrada inválida. Intente nuevamente.\n");
            limpiarBuffer();
            continue;
        }
        limpiarBuffer();
        
        if(opcion == 1) {
            char nombre[MAX_NOMBRE], tipo[MAX_TIPO];
            printf("\nIngrese nombre del cliente: ");
            fgets(nombre, MAX_NOMBRE, stdin);
            nombre[strcspn(nombre, "\n")] = '\0';
            
            printf("Ingrese tipo (General/VIP/Movilidad Reducida): ");
            fgets(tipo, MAX_TIPO, stdin);
            tipo[strcspn(tipo, "\n")] = '\0';
            
            if(sala->asientosLibres > 0) {
                // Buscar asiento libre
                Asiento* actual = sala->asientos;
                while(actual != NULL && strcmp(actual->estado, "Ocupado") == 0) {
                    actual = actual->siguiente;
                }
                
                if(actual != NULL) {
                    strcpy(actual->nombreCliente, nombre);
                    strcpy(actual->tipoCliente, tipo);
                    strcpy(actual->estado, "Ocupado");
                    sala->asientosLibres--;
                    
                    printf("¡Reserva exitosa! Asiento %d asignado.\n", actual->numero);
                }
            } else {
                printf("No hay asientos disponibles. Cliente agregado a la cola de espera.\n");
                if(sala->cola.cantidad < MAX_COLA) {
                    encolarCliente(&sala->cola, nombre, tipo, obtenerPrioridad(tipo));
                    printf("Hay %d clientes en espera.\n", sala->cola.cantidad);
                } else {
                    printf("La cola de espera está llena.\n");
                }
            }
        }
        else if(opcion == 2) {
            cancelarReserva(sala);
        }
    } while(opcion != 0);
}

void mostrarEstadoAsientos(Sala* sala) {
    printf("\n--- ESTADO DE ASIENTOS - SALA %d ---\n", sala->numeroSala);
    printf("Asientos libres: %d/%d\n", sala->asientosLibres, sala->totalAsientos);
    printf("Num\tEstado\tCliente\t\tTipo\n");
    
    Asiento* actual = sala->asientos;
    while(actual != NULL) {
        printf("%d\t%s\t%s\t%s\n", 
               actual->numero,
               actual->estado,
               actual->nombreCliente,
               actual->tipoCliente);
        
        actual = actual->siguiente;
    }
    pausa();
}

void cancelarReserva(Sala* sala) {
    char nombre[MAX_NOMBRE];
    printf("\nIngrese nombre del cliente a cancelar: ");
    fgets(nombre, MAX_NOMBRE, stdin);
    nombre[strcspn(nombre, "\n")] = '\0';
    
    bool encontrado = false;
    Asiento* actual = sala->asientos;
    
    while(actual != NULL && !encontrado) {
        if(strcmp(actual->nombreCliente, nombre) == 0) {
            strcpy(actual->estado, "Libre");
            strcpy(actual->nombreCliente, "");
            strcpy(actual->tipoCliente, "");
            sala->asientosLibres++;
            
            printf("Reserva cancelada. Asiento %d liberado.\n", actual->numero);
            encontrado = true;
            
            // Si hay clientes en espera, asignar el asiento
            if(sala->cola.cantidad > 0) {
                NodoCliente* cliente = desencolarCliente(&sala->cola, INT_MAX);
                if(cliente != NULL) {
                    strcpy(actual->nombreCliente, cliente->nombre);
                    strcpy(actual->tipoCliente, cliente->tipo);
                    strcpy(actual->estado, "Ocupado");
                    sala->asientosLibres--;
                    
                    printf("Asiento asignado a %s (%s) desde la cola de espera.\n", 
                           cliente->nombre, cliente->tipo);
                    
                    free(cliente);
                }
            }
        }
        
        actual = actual->siguiente;
    }
    
    if(!encontrado) {
        printf("No se encontró reserva para '%s'.\n", nombre);
    }
    pausa();
}

void gestionarColaEspera(Sala* sala) {
    printf("\n--- COLA DE ESPERA - SALA %d ---\n", sala->numeroSala);
    printf("Clientes en espera: %d\n", sala->cola.cantidad);
    
    if(sala->cola.cantidad > 0) {
        printf("\nPos.\tNombre\t\tTipo\tPrioridad\n");
        NodoCliente* actual = sala->cola.frente;
        int pos = 1;
        
        while(actual != NULL) {
            printf("%d\t%s\t%s\t%d\n", 
                   pos++,
                   actual->nombre,
                   actual->tipo,
                   actual->prioridad);
            actual = actual->siguiente;
        }
    }
    pausa();
}

void procesarFuncion(Sala* sala) {
    printf("\n--- PROCESAR FUNCIÓN - SALA %d ---\n", sala->numeroSala);
    printf("Asientos libres: %d\n", sala->asientosLibres);
    printf("Clientes en espera: %d\n", sala->cola.cantidad);
    
    if(sala->asientosLibres > 0 && sala->cola.cantidad > 0) {
        printf("\nAsignando asientos a clientes en espera...\n");
        
        while(sala->asientosLibres > 0 && sala->cola.cantidad > 0) {
            // Buscar asiento libre
            Asiento* asientoLibre = sala->asientos;
            while(asientoLibre != NULL && strcmp(asientoLibre->estado, "Libre") != 0) {
                asientoLibre = asientoLibre->siguiente;
            }
            
            if(asientoLibre == NULL) break;
            
            // Tomar cliente con mayor prioridad
            NodoCliente* cliente = desencolarCliente(&sala->cola, INT_MAX);
            if(cliente == NULL) break;
            
            // Asignar asiento
            strcpy(asientoLibre->nombreCliente, cliente->nombre);
            strcpy(asientoLibre->tipoCliente, cliente->tipo);
            strcpy(asientoLibre->estado, "Ocupado");
            sala->asientosLibres--;
            
            printf("Asiento %d asignado a %s (%s)\n", 
                   asientoLibre->numero, cliente->nombre, cliente->tipo);
            
            free(cliente);
        }
    }
    
    printf("\nFunción lista para comenzar!\n");
    pausa();
}

void guardarDatos(Sala* salas[]) {
    FILE* archivo = fopen("cine_data.dat", "wb");
    if(archivo == NULL) {
        printf("Error al abrir archivo para guardar datos\n");
        return;
    }
    
    // Guardar número de salas
    fwrite((int[]){TOTAL_SALAS}, sizeof(int), 1, archivo);
    
    for(int i = 0; i < TOTAL_SALAS; i++) {
        // Guardar datos de la sala
        fwrite(salas[i], sizeof(Sala), 1, archivo);
        
        // Guardar asientos
        Asiento* actual = salas[i]->asientos;
        while(actual != NULL) {
            fwrite(actual, sizeof(Asiento), 1, archivo);
            actual = actual->siguiente;
        }
        
        // Guardar cola de espera
        NodoCliente* cliente = salas[i]->cola.frente;
        while(cliente != NULL) {
            fwrite(cliente, sizeof(NodoCliente), 1, archivo);
            cliente = cliente->siguiente;
        }
    }
    
    fclose(archivo);
    printf("Datos guardados correctamente.\n");
}

void cargarDatos(Sala** salas) {
    FILE* archivo = fopen("cine_data.dat", "rb");
    if(archivo == NULL) {
        printf("No se encontraron datos previos. Inicializando sistema desde cero.\n");
        return;
    }
    
    int numSalas;
    fread(&numSalas, sizeof(int), 1, archivo);
    if(numSalas != TOTAL_SALAS) {
        printf("Advertencia: Número de salas no coincide. Inicializando sistema desde cero.\n");
        fclose(archivo);
        return;
    }
    
    for(int i = 0; i < TOTAL_SALAS; i++) {
        salas[i] = (Sala*)malloc(sizeof(Sala));
        if(salas[i] == NULL) {
            printf("Error al asignar memoria para sala\n");
            exit(EXIT_FAILURE);
        }
        
        // Leer datos de la sala
        fread(salas[i], sizeof(Sala), 1, archivo);
        salas[i]->asientos = NULL;
        salas[i]->cola.frente = NULL;
        salas[i]->cola.final = NULL;
        
        // Leer asientos
        Asiento* ultimoAsiento = NULL;
        for(int j = 0; j < salas[i]->totalAsientos; j++) {
            Asiento* nuevo = (Asiento*)malloc(sizeof(Asiento));
            fread(nuevo, sizeof(Asiento), 1, archivo);
            nuevo->siguiente = NULL;
            
            if(salas[i]->asientos == NULL) {
                salas[i]->asientos = nuevo;
            } else {
                ultimoAsiento->siguiente = nuevo;
            }
            ultimoAsiento = nuevo;
        }
        
        // Leer cola de espera
        NodoCliente* ultimoCliente = NULL;
        for(int k = 0; k < salas[i]->cola.cantidad; k++) {
            NodoCliente* nuevo = (NodoCliente*)malloc(sizeof(NodoCliente));
            fread(nuevo, sizeof(NodoCliente), 1, archivo);
            nuevo->siguiente = NULL;
            
            if(salas[i]->cola.frente == NULL) {
                salas[i]->cola.frente = nuevo;
            } else {
                ultimoCliente->siguiente = nuevo;
            }
            ultimoCliente = nuevo;
        }
        salas[i]->cola.final = ultimoCliente;
    }
    
    fclose(archivo);
    printf("Datos cargados correctamente.\n");
}

// ------------------------------
// Función principal
// ------------------------------
int main() {
    Sala* salas[TOTAL_SALAS] = {NULL};
    
    // Inicializar sistema con memoria dinámica
    cargarDatos(salas);
    if(salas[0] == NULL) {
        inicializarSistema(salas);
    }

    mostrarBienvenida();
    
    int opcion;
    
    do {
        mostrarMenuPrincipal();
        printf("Seleccione una opción: ");
        
        if(scanf("%d", &opcion) != 1) {
            printf("Entrada inválida. Intente nuevamente.\n");
            limpiarBuffer();
            continue;
        }
        limpiarBuffer();
        
        if(opcion >= 1 && opcion <= TOTAL_SALAS) {
            int salaSeleccionada = opcion - 1;
            printf("\n--- GESTIÓN SALA %d ---\n", salas[salaSeleccionada]->numeroSala);
            
            int subopcion;
            
            do {
                printf("\n1. Gestionar reservas\n");
                printf("2. Estado de asientos\n");
                printf("3. Cola de espera\n");
                printf("4. Procesar función\n");
                printf("0. Volver al menú principal\n");
                printf("Seleccione: ");
                
                if(scanf("%d", &subopcion) != 1) {
                    printf("Entrada inválida. Intente nuevamente.\n");
                    limpiarBuffer();
                    continue;
                }
                limpiarBuffer();
                
                switch(subopcion) {
                    case 1:
                        gestionarReservas(salas[salaSeleccionada]);
                        break;
                    case 2:
                        mostrarEstadoAsientos(salas[salaSeleccionada]);
                        break;
                    case 3:
                        gestionarColaEspera(salas[salaSeleccionada]);
                        break;
                    case 4:
                        procesarFuncion(salas[salaSeleccionada]);
                        break;
                    case 0:
                        break;
                    default:
                        printf("Opción inválida. Intente nuevamente.\n");
                }
            } while(subopcion != 0);
        } else if(opcion != 0) {
            printf("Opción inválida. Intente nuevamente.\n");
        }
    } while(opcion != 0);
    
    // Guardar datos y liberar memoria
    guardarDatos(salas);
    liberarSistema(salas);
    
    printf("\n¡Gracias por usar el sistema de gestión de cine!\n");
    return 0;
}
