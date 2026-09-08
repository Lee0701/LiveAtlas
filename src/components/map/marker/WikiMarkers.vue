<!--
  - Copyright 2022 Hyegyu Lee
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  - http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<script lang="ts">
import {defineComponent, computed, onMounted, watch, onUnmounted} from "vue";
import {useStore} from "@/store";
import LiveAtlasLayerGroup from "@/leaflet/layer/LiveAtlasLayerGroup";
import {
	LiveAtlasAreaMarker, LiveAtlasPointMarker, LiveAtlasMarkerSet,
	LiveAtlasWikiMarkerInfo
} from "@/index";
import {DynmapMarkerUpdate} from "@/dynmap";
import {
	createMarkerLayer, LiveAtlasMarkerType
} from "@/util/markers";
import {Layer} from "leaflet";
import axios from "axios";

const {apiEndpoint, categoryName} = window.wikiConfig;

export default defineComponent({
	props: {
		layerGroup: {
			type: Object as () => LiveAtlasLayerGroup,
			required: true
		}
	},

	setup(props) {
		const store = useStore(),
			currentMap = computed(() => store.state.currentMap),
			currentWorld = computed(() => store.state.currentWorld),
			layers = Object.freeze(new Map()) as Map<string, Layer>;

		let converter = currentMap.value!.locationToLatLng.bind(currentMap.value);

		const mapMarkers = async (offset = 0): Promise<any[]> => {
			const fields = ['Name', 'X', 'Y', 'Z', 'World', 'MinZoom', 'MaxZoom', 'Icon']
				.map((field) => '?' + field).join('|');
			const world = currentWorld.value!.name;
			const query = `[[${categoryName}]][[world::${world}]]|${fields}|offset=${offset}`;
			const params = {
				action: 'ask',
				query: query,
				format: 'json',
				origin: '*',
			}
			const resp = await axios.get(apiEndpoint, {params})
			const results = Object.values(resp.data.query.results);
			const queryContinueOffset = resp.data['query-continue-offset'];
			if(!queryContinueOffset) return results;
			const continued = await mapMarkers(queryContinueOffset);
			return [...results, ...continued];
		}

		const createMarker = (data: LiveAtlasWikiMarkerInfo) => {
			const {name, x, y, z, world, minZoom, maxZoom, icon, fullUrl} = data;
			const layer = createMarkerLayer({
				id: name,
				type: LiveAtlasMarkerType.POINT,
				location: {x, y, z},
				minZoom: minZoom,
				maxZoom: maxZoom,
				tooltip: name,
				iconUrl: icon,
				iconSize: [16, 16],
				popup: `<a href="${fullUrl}">${name}</a>`,
				isPopupHTML: true,
			} as LiveAtlasPointMarker, converter)
			layers.set(name, layer);
			props.layerGroup.addLayer(layer);
		}

		const createMarkers = () => {			
			mapMarkers().then(async (results) => {
				results.forEach((result) => {
					const {printouts} = result;
					createMarker({
						name: printouts['Name'][0],
						x: printouts['X'][0],
						y: printouts['Y'][0],
						z: printouts['Z'][0],
						world: printouts['World'][0],
						minZoom: printouts['MinZoom'][0],
						maxZoom: printouts['MaxZoom'][0],
						icon: printouts['Icon'][0],
						fullUrl: result.fullurl,
					});
				});
			})
		};

		const deleteMarker = (id: string) => {
			let marker = layers.get(id);

			if(!marker) {
				return;
			}

			props.layerGroup.removeLayer(marker);
			layers.delete(id);
		};

		onMounted(() => {
			createMarkers();
		});
	},

	render() {
		return null;
	}
});
</script>
