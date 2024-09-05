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
import {defineComponent, computed, onMounted, watch, onUnmounted} from "@vue/runtime-core";
import {useStore} from "@/store";
import LiveAtlasLayerGroup from "@/leaflet/layer/LiveAtlasLayerGroup";
import {LiveAtlasAreaMarker, LiveAtlasPointMarker, LiveAtlasMarkerSet} from "@/index";
import {DynmapMarkerUpdate} from "@/dynmap";
import {
	createMarkerLayer, LiveAtlasMarkerType
} from "@/util/markers";
import {Layer} from "leaflet";
import axios from "axios";

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
			layers = Object.freeze(new Map()) as Map<string, Layer>;

		let converter = currentMap.value!.locationToLatLng.bind(currentMap.value);

		const categoryMembers = async (cmcontinue = undefined) => {
			const params = {
				action: 'query',
				format: 'json',
				list: 'categorymembers',
				cmtitle: 'Category:지도 마커',
				cmcontinue,
			}
			const resp = await axios.get("https://wiki.kubecity.kr/api.php", {params})
			const {query} = resp.data;
			const result = query.categorymembers;
			if(!resp.data['continue']) return result;
			const continued = await categoryMembers(resp.data['continue'].cmcontinue);
			return [...result, ...continued];
		}

		const parse = async (page: string) => {
			const params = {
				action: 'parse',
				format: 'json',
				page,
			}
			const resp = await axios.get("https://wiki.kubecity.kr/api.php", {params})
			const {parse} = resp.data;
			return parse.text['*'];
		}

		const extractMarkerTag = (text: string) => {
			const div = document.createElement('div');
			div.innerHTML = text;
			const markerTag = div.querySelector('#map-marker');
			const data = JSON.parse(markerTag.innerText);
			return data;
		}

		const createMarker = (title, data) => {
			const {x, z, world, minzoom, maxzoom, icon} = data;
			const layer = createMarkerLayer({
				id: title,
				type: LiveAtlasMarkerType.POINT,
				location: {x, y: 64, z},
				minZoom: minzoom,
				maxZoom: maxzoom,
				tooltip: title,
				iconUrl: icon,
				iconSize: [16, 16],
			} as LiveAtlasPointMarker, converter)
			layers.set(title, layer);
			props.layerGroup.addLayer(layer);
		}

		const createMarkers = () => {			
			categoryMembers().then(async (members) => {
				const markers = await Promise.all(members.map(async (member) => {
					const text = await parse(member.title);
					const data = extractMarkerTag(text);
					createMarker(member.title, data);
				}));
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
