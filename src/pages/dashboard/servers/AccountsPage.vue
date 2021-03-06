<template>
	<div class="dashboard">
		<div class="inner-dashboard">
			<div class="dashboard-content" v-if="!isLoading">
				<div class="dashboard-header">
					<h1>Serveurs</h1>
					<ul class="actions">
						<li>
							<a class="btn add-user" @click.prevent="showModal">
								<i class="icon ion-ios-add-circle"></i>Ajouter
								un utilisateur
							</a>
						</li>
					</ul>
				</div>
				<div class="dashboard-section">
					<modal v-show="isModalVisible" @close="closeModal">
						<template #header>
							<div>
								Ajouter un utilisateur
							</div>
						</template>
						<template #body>
							<form class="add-user-form">
								<div class="field">
									<label class="label"
										>Nom de l'utilisateur</label
									>
									<input
										class="input"
										:class="{
											'has-error': errors.username
										}"
										required
										v-model="fields.username"
										type="text"
										@keydown="errors.username = ''"
									/>
									<p class="help error">
										{{ errors.username }}
									</p>
								</div>
								<div class="field">
									<label class="label">Mot de passe</label>
									<input
										class="input"
										:class="{
											'has-error': errors.password
										}"
										required
										v-model="fields.password"
										type="password"
										@keydown="errors.password = ''"
									/>
									<p class="help error">
										{{ errors.password }}
									</p>
								</div>
							</form>
						</template>
						<template #footer>
							<div>
								<button
									class="btn cancel"
									@keyup.27="closeModal"
									@click="closeModal"
								>
									Annuler
								</button>
								<button
									@click.prevent="addUser"
									class="btn submit"
									:disabled="addPending"
								>
									<span v-if="addPending === false">
										Ajouter
									</span>

									<Loader
										v-else
										:strokeColor="'#fff'"
										:width="'20'"
										:height="'20'"
									></Loader>
								</button>
							</div>
						</template>
					</modal>
					<div>
						<table class="table table-dashboard">
							<thead>
								<tr>
									<th scope="col">UID</th>
									<th scope="col">GID</th>
									<th scope="col">Login</th>
									<th scope="col">Nom utilisateur</th>
									<th scope="col">Répertoire</th>
								</tr>
							</thead>
							<tbody>
								<tr
									v-for="(item, index) in this.infos.users"
									v-bind:key="index"
								>
									<td>{{ item.Uid }}</td>
									<td>{{ item.Gid }}</td>
									<td>{{ item.Username }}</td>
									<td>
										{{ item.Name }}
									</td>
									<td>{{ item.HomeDir }}</td>
								</tr>
							</tbody>
						</table>
					</div>
				</div>
			</div>
			<Loader
				v-else
				:strokeColor="'#2873ed'"
				:width="'35'"
				:height="'35'"
			></Loader>
		</div>
	</div>
</template>

<script>
import {
	debug,
	parseToObject,
	isArraysEqual,
	serverGetters,
	isFieldValid
} from "../../../utils";
import Loader from "@/components/ui/loader";
import Modal from "@/components/partials/Modal";
import axios from "axios";
import {apiUrl} from "../../../config";

export default {
	name: "ServerAccountsPage",
	data() {
		return {
			infos: {},
			isLoading: true,
			errors: {},
			isModalVisible: false,
			addPending: false,
			isUserAdded: false,
			fields: {
				username: "",
				password: ""
			}
		};
	},
	components: {
		Loader,
		Modal
	},
	methods: {
		retrieveInfos() {
			this.$socket.sendObj(serverGetters.getUsers());
		},
		showModal() {
			this.isModalVisible = true;
		},
		closeModal() {
			this.resetForm();
			this.isModalVisible = false;
		},
		closeOnEscape(e) {
			const key = e.which || e.keyCode || e.detail;
			if (key === 27 && this.isModalVisible === true) {
				this.closeModal();
			}
		},
		addUser() {
			this.errors = {};
			for (const field in this.fields) {
				if (isFieldValid(this.fields[field])) {
					this.errors[field] =
						"Ce champs et requis et doit être valide";
				}
			}
			if (Object.keys(parseToObject(this.errors)).length === 0) {
				this.addPending = true;
				this.$socket.sendObj(
					serverGetters.addUser(
						this.fields.username,
						this.fields.password
					)
				);
				const userAdded = setInterval(() => {
					if (this.isUserAdded === true) {
						this.isUserAdded = false;
						this.addPending = false;
						debug("success", "user added", this.isUserAdded);
						this.closeModal();
						this.retrieveInfos();
						this.refreshData();
						this.resetForm();
						clearInterval(userAdded);
					}
				}, 100);
			}
		},
		resetForm() {
			this.errors = {};
			for (const field in this.fields) {
				this.fields[field] = "";
			}
		},
		fetchData(serverId) {
			this.isLoading = true;
			axios
				.get(`${apiUrl}servers/${serverId}`, {
					headers: {
						"x-access-token": this.token
					}
				})
				.then(response => {
					debug("info", "fetchData -> response", response.data);
					this.server = response.data;
					this.setSockets();
				});
		},
		setSockets() {
			const host = parseToObject(this.server).host;
			const port = parseToObject(this.server).port;
			this.$store.commit("SET_SOCKET_URL", `ws://${host}:${port}/ws/v1`);
			const checkInfos = setInterval(() => {
				const requiredFields = ["users"];
				const infosKeys = Object.keys(parseToObject(this.infos));
				if (isArraysEqual(infosKeys, requiredFields)) {
					debug(
						"success",
						"checkInfos -> !isLoading",
						isArraysEqual(infosKeys, requiredFields)
					);
					this.isLoading = false;
					clearInterval(checkInfos);
				}
			}, 100);
		},
		refreshData() {
			this.fetchData(this.$route.params.id);
		}
	},
	created() {
		window.addEventListener("keyup", this.closeOnEscape);
		debug("info", "mounted -> this.dataServer", this.dataServer);
		if (this.dataServer !== undefined) {
			this.server = this.dataServer;
			debug("info", "mounted -> this.server", this.server);
			this.setSockets();
		} else {
			if (
				this.$store.getters.token === undefined ||
				this.$store.getters.token === ""
			) {
				return;
			}
			this.token = this.$store.getters.token;
			this.currentUserId = this.$store.getters.userId;
			this.fetchData(this.$route.params.id);
		}
	},
	beforeMount() {
		const initConnection = setInterval(() => {
			if (this.$store.getters.isConnected === true) {
				this.retrieveInfos();
				this.$socket.onmessage = msg => {
					const parsedData = JSON.parse(msg.data);
					if (
						parsedData.data &&
						!parsedData.error &&
						parsedData.event.value === "adduser" &&
						this.isUserAdded === false
					) {
						debug("info", "data -> this.infos", parsedData);
						this.isUserAdded = true;
					} else {
						Object.assign(this.infos, {
							[`${parsedData.event.value}`]: parsedData.data
						});
						debug(
							"info",
							"data -> this.infos",
							parseToObject(this.infos)
						);
					}
				};
				clearInterval(initConnection);
			}
		}, 100);
	}
};
</script>

<style lang="scss" scoped>
.dashboard {
	.inner-dashboard {
		.dashboard-content {
			.dashboard-header {
				display: flex;
				align-items: center;
				justify-content: space-between;
				margin-bottom: 30px;

				h1 {
					color: #303133;
					margin-top: 0;
					margin-bottom: 30px;
				}

				.actions {
					padding: 0;
					margin: 0;
					list-style: none;

					.add-user {
						position: relative;
						font-size: 14px;
						padding: 12px;
						display: flex;
						align-items: center;
						justify-content: center;

						> .icon {
							font-size: 18px;
							margin-right: 10px;
						}
					}
				}
			}
			form {
				.field {
					margin-bottom: 20px;
					textarea {
						height: 100px;
						min-height: 100px;
						width: 100%;
					}
				}
			}

			.modal {
				form {
					&.add-user-form {
						padding: 0 20px;

						.field {
							position: relative;

							* {
								display: block;
								width: 100%;
								margin-bottom: 10px;
							}

							.input {
								background: #fcfcfc;
								border: 2px solid #e7e7e7;
								margin: 10px auto;
								padding: 10px 15px;
								width: 100%;
								display: block;
								border-radius: 3px;
								font-size: 16px;
								transition: border-color ease-in-out 0.15s,
									box-shadow ease-in-out 0.15s;
								text-align: left;
								height: auto;
								-webkit-appearance: none;
								outline: none;

								&:hover {
									border-color: rgba(39, 117, 237, 0.23);
								}

								&:focus {
									border-color: #2874ed;
								}

								&.has-error {
									border-color: #ff3860;
								}
							}

							p.help {
								margin: 0;
								position: absolute;
								bottom: -15px;
								font-size: 12px;

								&.error {
									color: #ff3860;
								}
							}
						}
					}
				}

				.modal-footer {
					> div {
						display: flex;

						button {
							padding: 12px 20px;

							&:first-child {
								margin-right: 10px;
							}

							&.cancel {
								color: #303133;
								background: transparent;

								&:hover {
									background: #e7e7e7;
								}
							}

							&.submit {
								min-width: 90px;
							}
						}
					}
				}
			}
		}
	}
}
</style>
