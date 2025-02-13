<template>
    <v-container fluid class="pa-0">
        <v-card flat>
            <v-card-title class="custom-title text-h4 font-weight-bold white--text pb-4">
                <h1>Mi Cuenta</h1>
            </v-card-title>

            <v-row no-gutters>
                <v-col cols="12" md="3">
                    <v-card flat class="h-100">
                        <v-list>
                            <v-list-item lines="two" :prepend-avatar="require('@/assets/icon-account.png')"
                                :title="fullName" :subtitle="email" class="avatar-item">
                            </v-list-item>
                        </v-list>
                        <v-divider></v-divider>
                        <v-list nav density="compact">
                            <v-list-item prepend-icon="mdi-view-dashboard" title="Mi Cuenta" value="home"
                                to="/miCuenta"></v-list-item>
                            <v-list-item prepend-icon="mdi-forum" title="Historial" value="about"
                                to="/historyTranscriptor"></v-list-item>
                            <v-list-item prepend-icon="mdi mdi-credit-card" title="Mis Suscripciones" value="about"
                                to="/miSuscripcion"></v-list-item>
                            <v-list-item prepend-icon="mdi-logout" :title="isAuthenticated ? 'Cerrar Sesión' : ''"
                                @click="handleAuthAction" to="/" id="authButton" class="error--text"></v-list-item>
                        </v-list>
                    </v-card>
                </v-col>

                <v-col cols="12" md="9">
                    <v-card flat class="pa-6">
                        <v-list>
                            <v-list-item>
                                <v-list-item-title class="text-h6">Nombre</v-list-item-title>
                                <v-list-item-subtitle>{{ fullName }}</v-list-item-subtitle>
                            </v-list-item>

                            <v-list-item>
                                <v-list-item-title class="text-h6">Nombre de Usuario</v-list-item-title>
                                <v-list-item-subtitle>{{ userName }}</v-list-item-subtitle>
                            </v-list-item>

                            <v-list-item>
                                <v-list-item-title class="text-h6">Correo</v-list-item-title>
                                <v-list-item-subtitle>{{ email }}</v-list-item-subtitle>
                            </v-list-item>

                            <v-list-item>
                                <v-list-item-title class="text-h6">Número</v-list-item-title>
                                <v-list-item-subtitle>{{ phoneNumber }}</v-list-item-subtitle>
                            </v-list-item>
                        </v-list>
                        <!-- URL  -->
                        <v-btn x-large color="#42b983" dark class="mt-8"
                            :href="`http://34.176.135.227:8081/realms/Transcriptor/account/`" target="_blank">
                            Editar
                        </v-btn>

                    </v-card>
                </v-col>
            </v-row>
        </v-card>
    </v-container>
</template>

<script>
import keycloak from '@/keycloak';
import config from "@/config";

export default {
    name: 'MiCuenta',
    data() {
        return {
            fullName: '',
            userName: '',
            email: '',
            phoneNumber: '',
            isAuthenticated: false,
        };
    },
    methods: {
        handleAuthAction() {
            if (this.isAuthenticated) {
                keycloak.logout({
                    redirectUri: window.location.origin
                });
            } else {
                keycloak.login();
            }
        },
        async get_user_data() {
            if (keycloak.authenticated) {
                const token = keycloak.tokenParsed;

                this.fullName = token.given_name || '';  // Se asume que el nombre está en given_name
                this.userName = token.preferred_username || '';
                this.email = token.email || '';

                // Propiedades personalizadas
                const phonePrefix = token.prefijo || '';
                const phoneNumber = token.telefono || '';
                this.phoneNumber = `${phonePrefix} ${phoneNumber}`.trim();

                // Ahora verificamos si los datos del usuario en Keycloak coinciden con los datos almacenados
                const userData = {
                    id_usuario: token.sub,
                    nombre: token.given_name || '',
                    apellido: token.family_name || '',
                    prefijo: token.prefijo || '',
                    numero_telefono: token.telefono ? parseInt(token.telefono) : null,
                    email: token.email || '',
                    username: token.preferred_username || '',
                };

                try {
                    const response = await fetch(`${config.BASE_URL}:8000/usuarios/${userData.id_usuario}`);

                    if (response.status === 404) {
                        // Usuario no existe, lo guardamos
                        await fetch(`${config.BASE_URL}:8000/usuarios`, {
                            method: 'POST',
                            headers: {
                                'Content-Type': 'application/json',
                            },
                            body: JSON.stringify(userData),
                        });

                    } else if (response.ok) {
                        // Usuario existe, verificamos cambios
                        const userInDb = await response.json();
                        const hasChanges = this.checkForChanges(userInDb, userData);

                        if (hasChanges) {
                            await fetch(`${config.BASE_URL}:8000/usuarios/${userData.id_usuario}`, {
                                method: 'PUT', 
                                headers: {
                                    'Content-Type': 'application/json',
                                },
                                body: JSON.stringify(userData),
                            });
                        } 
                    } else {
                        throw new Error('Error al verificar el usuario en la base de datos');
                    }
                } catch (error) {
                    console.error('Error:', error);
                }
            }
        },
        checkForChanges(userInDb, userData) {
            // Compara cada campo relevante
            return (
                userInDb.nombre !== userData.nombre ||
                userInDb.apellido !== userData.apellido ||
                userInDb.prefijo !== userData.prefijo ||
                userInDb.numero_telefono !== userData.numero_telefono ||
                userInDb.email !== userData.email ||
                userInDb.username !== userData.username
            );
        },
    },
    mounted() {
        // Verificar el estado de autenticación al montar el componente
        this.isAuthenticated = keycloak.authenticated;
        keycloak.onAuthLogout = () => {
            this.isAuthenticated = false;
        };
        this.get_user_data();
    }
};
</script>

<style scoped>
.v-card {
    border-radius: 8px;
}

.v-list-item--active {
    background-color: #e6f7f0;
}

#authButton {
    font-size: 14px;
    color: #ff5252;
}

.v-img {
    background-color: #1ebea4;
}

.v-list-item-title {
    font-weight: bold;
    color: #42b983;
}

.v-list-item-subtitle {
    color: rgba(0, 0, 0, 0.6);
    font-weight: bold;
}

.custom-title {
    background-color: #1ebea4;
    width: 100%;
    height: 200px;
    display: flex;
    /* Asegura que el título esté centrado */
    align-items: center;
    justify-content: center;
    color: white;
}
</style>