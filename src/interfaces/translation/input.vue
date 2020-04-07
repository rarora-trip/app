<template>
	<v-sheet v-if="relationshipSetup">
		<v-select
			v-model="currentLanguage"
			class="language-picker"
			:options="options.languages"
			icon="translate"
		/>

		<hr />

		<div v-if="loading === false && initialValues !== null" class="body">
			<v-form
				full-width
				:key="currentLanguage"
				:collection="relation.collection_many.collection"
				:fields="translatedFields"
				:values="currentLanguageValues"
				:primary-key="existing && existing[translationsCollectionPrimaryKeyField]"
				@stage-value="saveLanguage"
			/>
		</div>

		<v-spinner v-else />
	</v-sheet>

	<v-notice v-else color="warning" icon="warning">
		{{ $t('relationship_not_setup') }}
	</v-notice>
</template>

<script>
import mixin from '@directus/extension-toolkit/mixins/interface';
import { mapValues, clone, find, merge } from 'lodash';

export default {
	mixins: [mixin],
	data() {
		return {
			currentLanguage: Object.keys(this.options.languages)[0],
			loading: false,
			initialValues: null,
			relationalChanges: []
		};
	},
	computed: {
		translatedFields() {
			if (this.relationshipSetup === false) {
				return;
			}

			return mapValues(this.relation.collection_many.fields, field => {
				field = clone(field); // remove vue reactivity

				// Prevent updating the recursive relational key
				if (field.field === this.relation.field_many.field) {
					field.readonly = true;
				}

				return field;
			});
		},
		defaults() {
			return mapValues(clone(this.translatedFields), f => f.default_value);
		},
		existing() {
			return find(this.initialValues, {
				[this.options.languageField]: this.currentLanguage
			});
		},
		translationsCollectionPrimaryKeyField() {
			return Object.values(this.relation.collection_many.fields).find(
				field => field.primary_key === true
			).field;
		},
		currentLanguageValues() {
			const existingChanges = find(this.relationalChanges, {
				[this.options.languageField]: this.currentLanguage
			});

			return merge({}, this.existing || this.defaults, existingChanges);
		},
		relationshipSetup() {
			return !!this.relation?.collection_many;
		},
		currentPrimaryKey() {
			const { field } = find(this.fields, { primary_key: true });
			return this.values[field];
		}
	},
	watch: {
		relationalChanges: {
			deep: true,
			handler(value) {
				if (value) {
					this.$emit('input', value);
				}
			}
		}
	},
	created() {
		this.fetchInitial();
	},
	methods: {
		saveLanguage({ field, value }) {
			const existingChanges = find(this.relationalChanges, {
				[this.options.languageField]: this.currentLanguage
			});
			if (existingChanges) {
				this.relationalChanges = this.relationalChanges.map(update => {
					if (update[this.options.languageField] === this.currentLanguage) {
						return merge({}, update, { [field]: value });
					}

					return update;
				});
			} else {
				const update = {
					[field]: value,
					[this.options.languageField]: this.currentLanguage
				};
				if (this.existing) {
					const primaryKeyField = find(this.translatedFields, { primary_key: true })
						.field;
					const relatedPrimaryKey = this.existing[primaryKeyField];
					update[primaryKeyField] = relatedPrimaryKey;
				}
				this.relationalChanges = [...this.relationalChanges, update];
			}
		},

		// VIATOR- updateTempStore is method used to get initialValues from store.
		// As directus uses backend db to return values and we need values that were staged in state of vue.
		updateTempStore(value) {
			var manyField = this.fields.id.collection;
			if(manyField === 'articles_sections' || manyField === 'shelf_collections_sections') {
				manyField = 'sections';
			}
			const storedSections=this.$store.state.edits.values[manyField];
			if(storedSections && Array.isArray(value)) {
				storedSections.map(section => {
					if(section.content_translations) {
						section.content_translations.map(storedTranslation => {
							if(storedTranslation.id ) {
								const inputData = value.find(x => x.id === storedTranslation.id);
									if(inputData) {
										const translation = merge(inputData,storedTranslation);
									}

							} else {
								if(section.internal_temp_id === this.currentPrimaryKey )
								{
									value.push(storedTranslation);
								}
							}
						});
					}
					});
			}
		},
		async fetchInitial() {
			if (this.relationshipSetup === false) {
				return;
			}

			if (false) {
				this.initialValues = [];
				return;
			}

			this.loading = true;
			const { collection } = this.relation.collection_many;
			const { field } = this.relation.field_many;
			var { data } = await this.$api.getItems(collection, {
				filter: {
					[field]: {
						eq: this.currentPrimaryKey
					}
				}
			});
			//VIATOR- before returning initial values to form, we fetch values from store and merge it with values received from backend.
			this.updateTempStore(data);
			this.initialValues = data;
			this.loading = false;
		}
	}
};
</script>

<style lang="scss" scoped>
.language-picker {
	margin-bottom: 24px;
}

hr {
	border: none;
	border-bottom: 2px solid var(--input-border-color);
	border-radius: 1px;
	margin-bottom: 24px;
}

.body {
	--form-vertical-gap: 24px;
	--form-horizontal-gap: 12px;
	--type-label-size: 15px;
	--input-height: 44px;
	--input-font-size: 14px;
	--input-label-margin: 4px;
	--input-background-color-alt: var(--input-background-color);
}
</style>
