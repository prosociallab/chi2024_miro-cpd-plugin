<script>
	import Tab, { Label } from "@smui/tab";
	import TabBar from "@smui/tab-bar";
	import Button from "@smui/button";
  	import Textfield from '@smui/textfield';
	import { onMount } from "svelte";
	import { Tooltip, Form, FormText, Input, InputGroup, InputGroupText } from "sveltestrap";
	import Pluralize from 'pluralize';
	import { Configuration, OpenAIApi } from 'openai';
	import CircularProgress from '@smui/circular-progress';
	import Paper, {Title, Subtitle, Content} from '@smui/paper';
	import Select, {Option} from '@smui/select';
	import { OpenAIClient, AzureKeyCredential } from '@azure/openai';
    import { text } from "svelte/internal";

	const baseurl = "https://prosocial2.openai.a"
	const basecred = "fbe11d32db74476cb0"

	const client = new OpenAIClient(baseurl+"zure.com/", new AzureKeyCredential(basecred + "8b540b4c8" + "6ed8c"));
    const deploymentId = "gpt-4-proso";

	const add = "smiCEz1HPnjQZdQ"+"54"

	

	const configuration = new Configuration({
		apiKey: "sk-2Q5TJeDm6eq4WGUaXRjcT3BlbkFJ"+add+"eoi"
	});
	delete configuration.baseOptions.headers['User-Agent'];
	const openai = new OpenAIApi(configuration);

	const getRecommendations = async function (inst, input) {
		const events = client.listChatCompletions(deploymentId+"cial", [{role: "system", content: inst}, {role: "user", content: input}], { maxTokens: 256 });
		let txt = "";
		for await (const event of events) {
			for (const choice of event.choices) {
				if (choice.delta.content != null) {
					txt += choice.delta.content;
				}
			}
		}
		return txt;
	}

	const getRecommendations_openai = async function (inst, input) {
		const chatCompletion = await openai.createChatCompletion({
			model: "gpt-4",
			messages: [{role: "system", content: inst}, {role: "user", content: input}],
			temperature: 1,
			max_tokens: 256,
			top_p: 1,
			frequency_penalty: 0,
			presence_penalty: 0,
		});
		return chatCompletion.data.choices[0].message["content"];
	}

	var inProgress = false;
	var response_array = [];
	var recommending = true;


	// let distal_input_rec = "";
	// let barrier_input_rec = "";
	// let proximal_input_rec = "";
	// let strategy_input_rec = "";
	// let mechanism_input_rec = "";
	// let moderator_input_rec = "";
	// let precondition_input_rec = "";
	// var distal=true;
	// var barrier=true;
	// var proximal=true;
	// var strategy=true;
	// var mechanism=true;
	// var moderator=true;
	// var precondition=true;
	


	let distal_input = "";
	let barrier_input = "";
	let proximal_input = "";
	let strategy_input = [""];
	let mechanism_input = [""];
	let moderator_input = "";
	let precondition_input = "";
	let prev_comp="NA";
	let prev_input="";
	let next_comp="NA";
	let next_input="";

	$: questions = [
		"1. What is your desired implementation outcome?",
		`2. What is the obstacle that is getting in the way of achieving the desired outcome (${responses[0].toLowerCase()})?`,
		"3. What is the earliest signs of change?",
		`4. What is the strategy that will help overcome the barrier (${responses[1].toLowerCase()})?`,
		`5. How or why would the proposed strategy work to resolve the barrier (${responses[1].toLowerCase()})?`
	];
	let questions_instruction = [
		"Desired implementation outcome",
		"The obstacle that is getting in the way of achieving the desired outcome",
		"The earliest signs of change that is followed by the desired implementation outcome",
		"The strategy that will help overcome the barrier",
		"How or why the proposed strategy would work to resolve the barrier",
	];
	let entities = [
		"Distal implementation outcome",
		"Barrier",
		"Proximal outcome",
		"Implementation strategy",
		"Mechanism"
	];
	let CPD_def;
	

	let responses = [
		"",
		"",
		"",
		"",
		""
	]
	let index = 0;
	let component_names = [
		"Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"
	];
	let default_comp = "Strategy"
	// let component_need_recommend=[];
	let prev_list = ["NA","Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"]
	let next_list= ["NA","Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"]


	let feedback="";
	let component_list;
	let content_list;
	let connection_list;
	let id_list;
	let coordinates;
	let precond_list;
	let mod_list;
	var not_all_req=false;
	var comp_extra=false;
	var multiple_conn=false;
	var miss_conn=false;
	var not_corr_conn=false;
	var wrong_coord=false;
	var basic_conn = false;

	var feedback_arr = [];
	var incorr_link=[];
	var faq_feedback;
	var faq_def;
	
	
	var arr;
	let faq_names = [
		"Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome",
		"Precondition",
		"Moderator"
	]
	var faq_sel = false;
	let faq_definitions= [
		"Strategy an element that the diagram is intended to unpack. It is important to make the strategy concrete, to write it as it would be performed in that particular setting.",
		"A mechanism is the causal process that a performance of the strategy is intended to put in motion, through which the strategy is meant to resolve a barrier. In other words, the mechanism is how and why a strategy works to address the barrier. A strategy can have several mechanisms through which it operates, but it must successfully activate at least one.",
		"A barrier is the obstacle that is getting in the way of achieving the desired outcome. There may be multiple barriers impeding weight gain, which may eventually require multiple strategies or an adjustment of the strategy to target multiple barriers. As a first step, each barrier should be considered independently.",
		"We refer to the most direct result of the operation of a mechanism, which, in turn, influences other outcomes further downstream, as the “proximal outcome” (terms “near-term” or “immediate outcomes” are also used).",
		"A distal outcome is the motivation for the use of the strategy. It is a result of the whole chain of events that precedes it, and it’s the ultimate success criterion. It is likely an implementation outcome such as adoption, fidelity, or sustainment but may also be a clinical outcome. The extent to which this chain of events—starting with an application of a strategy and finishing with the desired distal outcome—successfully takes place depends on precondition and moderator.",
		"A precondition is a factor that is necessary for a part of the causal process to take place. If a precondition is not met, the part of the causal process to which it applies will break, stopping the flow of influence.",
		"Some factors, when they are present, either facilitate or impede a part of the causal process. Drawing on statistical terminology, we call such factors “moderators” or “effect modifiers.” Like preconditions, moderators can function in different places along the causal chain. The presence and level of a particular factor either helps or hurts the operation of the causal process."
	]

	let box
	let xLeft = 0
	let xScroll = 0
	let xWidth = 0
	let yTop = 0
	let yScroll = 0
	let yHeight = 0
	function parseScroll() {
		/*
		xLeft=box.scrollLeft
		xScroll=box.scrollWidth
		xWidth=box.clientWidth
		yTop=box.scrollTop
		yHeight=box.clientHeight
		yScroll=box.scrollHeight
		*/
	}

	function restart_wizard() {

		responses = [
		"",
		"",
		"",
		"",
		""
		]
		index = 0;
		response_array=[]
		return
	}
	
	function initialize_definitions() {
		CPD_def = new Map()
		CPD_def.set("Distal outcome", "Desired implementation outcome")
		CPD_def.set("Proximal outcome", "The earliest signs of change that is followed by the desired implementation outcome")
		CPD_def.set("Barrier", "The obstacle that is getting in the way of achieving the desired outcome",)
		CPD_def.set("Mechanism", "How or why the proposed strategy would work to resolve the barrier")
		CPD_def.set("Strategy", "The strategy that will help overcome the barrier")
	}
	async function checkCPD() {
		feedback=""
		feedback_arr=[];
		not_all_req=false;
		comp_extra=false;
		multiple_conn=false;
		not_corr_conn=false;
		miss_conn=false;
		wrong_coord=false;
		id_list = new Map();
		connection_list = new Map();
		coordinates = new Map();
		precond_list = new Map();
		mod_list = new Map();
		const items = await miro.board.getSelection();
		component_list = new Map();
		component_list.set("Strategy", [])
		component_list.set("Mechanism", [])
		component_list.set("Barrier", [])
		component_list.set("Proximal outcome", [])
		component_list.set("Distal outcome", [])
		content_list = new Map();
		var connection_main = new Map();

		items.forEach((item) => {
			if (item.type != "connector") {
				coordinates.set(item.id, [item.x, item.y, item.width, item.height])
				content_list.set(item.id, item.content)
				if (is_strategy(item.shape)) {
					set_component(component_list,"Strategy", item.id)
					// component_list.set(item.id, "Strategy")
					id_list.set(item.id, "Strategy")
				}
				if (is_mechanism(item.shape)) {
					set_component(component_list,"Mechanism", item.id)
					id_list.set(item.id, "Mechanism")
				}
				if (is_barrier(item.shape)) {
					set_component(component_list,"Barrier", item.id)
					id_list.set(item.id, "Barrier")
				}
				if (is_proximal(item.shape, item.content)) {
					set_component(component_list,"Proximal outcome", item.id)
					id_list.set(item.id, "Proximal outcome")
				}
				if (is_distal(item.shape, item.content)) {
					set_component(component_list,"Distal outcome", item.id)
					id_list.set(item.id, "Distal outcome")
				}
				// If precondition
				if (is_precondition(item.shape)) {
					precond_list.set(item.id, [item.x, item.y, item.width, item.height])

					coordinates.delete(item.id)
					// id_list.delete()
				}
				if (is_moderator(item.shape)) {
					mod_list.set(item.id, [item.x, item.y, item.width, item.height])
					coordinates.delete(item.id)
				}
			}
		});
			
		items.forEach((item) => {
		
			if (item.type=="connector") {
				
					
				var start = item.start
				var end = item.end
				if (!start || (!end && !precond_list.has(start.item)) || (!end && !mod_list.has(start.item))) {
					miss_conn=true;
					feedback="There might be missing connections between components selected. Please make sure all your components are connected."
					return;
				}

				start = start.item
				end = end.item
				if (!connection_list.has(start)) {

					var temp = new Map()
					var n = new Array()
					var p = new Array()
					n.push(end)
					temp.set("prev", p)
					temp.set("next", n)
					connection_list.set(start, temp)

				} else {
					var temp = connection_list.get(start)
					var n;
					n = temp.get("next")
					n.push(end)
				
					temp.set("next", n)
					
					connection_list.set(start, temp)
				}
				if (!connection_list.has(end)) {
					var temp = new Map()
					var p = new Array()
					var n = new Array()
					p.push(start)
					temp.set("prev", p)
					temp.set("next", n)
					connection_list.set(end, temp)
				} else {
					var temp = connection_list.get(end)
					var p = temp.get("prev")
					p.push(start)

					connection_list.set(end, temp)
				}
			}
		});



		for (let [key, val] of connection_list) {
			if (id_list.has(key)) {
				
				connection_main.set(key, val)
			} 
		}
		// Check if the CPD contains all the required components 
		if (!req_comp_check(component_list)) {
			not_all_req=true;
			feedback="You might consider adding the following components:"
			return;
		}

		if (!single_conn(connection_main, id_list)) {
			multiple_conn = true;
			return;
		}
		
		if (!connectedness(connection_main, id_list)) {
			feedback="There are some potential issues with the CPD:"
			not_corr_conn =true;
			return
		}
		if (!basic_placement(connection_main, id_list)) {
			feedback="There are some potential issues with the CPD:"
			basic_conn=true;
			return
		}
		if (!all_conn(connection_main, id_list)) {
			miss_conn= true;
			feedback="There might be missing connections between components selected. Please make sure all your components are connected."
			return 
		}
		

		not_all_req=true;
		feedback="No issues with your CPD pathway!"

	}
	function connectedness(connections, ids) {
		feedback_arr=[]
		for (let [comp, conn] of connections) {
			var prev = conn.get("prev")
			var next = conn.get("next")
			if (ids.get(comp) =="Strategy" && next.length !=0) {
				if (ids.get(next[0])=="Strategy") {
					feedback_arr.push("There should not be a Strategy after a Strategy")
				} else if (ids.get(next[0])!="Mechanism") {
					feedback_arr.push("There should be a Mechanism following a Strategy instead of " + ids.get(next[0]))
				}
			} else if (ids.get(comp) =="Strategy" && next.length ==0) {
				feedback_arr.push("There should be a Mechanism following a Strategy")
			}
			if (ids.get(comp)=="Barrier" && next.length!=0) {
				if (ids.get(next[0]) =="Barrier") {
					feedback_arr.push("There should not be a Barrier after a Barrier")
				} else if (ids.get(next[0])=="Strategy") {
					feedback_arr.push("There should not be a Strategy after a Barrier")
				} else if (ids.get(next[0])!="Proximal outcome") {
					feedback_arr.push("There should be a Proximal outcome following a Barrier")
				}
			} 
		}
		if (feedback_arr.length!=0) {
			return false
		}
		return true
	}
	function basic_placement(connections, ids) {
		feedback_arr=[]
		for (let [comp, conn] of connections) {
			var prev = conn.get("prev")
			var next = conn.get("next")
			if (ids.get(comp) =="Strategy" && prev.length!=0) {
				feedback_arr.push("Strategy should be placed at the start of the CPD")
			}
			if (ids.get(comp)=="Distal outcome" && next.length!=0) {
				feedback_arr.push("Distal outcome should be placed at the end of the CPD")
			}
		}
		if (feedback_arr.length!=0) {
			return false
		}
		return true
		
	}
	function x_placement(connections, coord) {
		for (let [comp, conn] of connections) {
			var next = conn.get("next")[0]
			var curr_coord = coord.get(comp)
			var next_coord = coord.get(next)
			if (next_coord) {
				var next_next = connections.get(next).get("next")[0]
				if (curr_coord[0] + curr_coord[2] > next_coord[0] && next_coord[0] + next_coord[2] < coord.get(next_next)[0]) {
					return false;
				} 
			} 
		}
		return true;
	}


	function corr_link(ideal, connections, ids) {
		for (let [comp, conn] of connections) {
			var next = conn.get("next")[0]
			var temp = ideal.get(comp)
			if (temp != next) {
				incorr_link.push(ids.get(comp)+": connected to " + ids.get(next) + ", should be connected to " + ids.get(temp))
			}
		
		}
		if (incorr_link.length > 0) {
			return false;
		}
		return true;
	}

	function create_ideal_map(comp) {
		let ideal_mapping = new Map()
		ideal_mapping.set(comp.get("Strategy")[0], comp.get("Mechanism")[0])
		ideal_mapping.set(comp.get("Mechanism")[0], comp.get("Barrier")[0])
		ideal_mapping.set(comp.get("Barrier")[0], comp.get("Proximal outcome")[0])
		ideal_mapping.set(comp.get("Proximal outcome")[0], comp.get("Distal outcome")[0])
		return ideal_mapping
	}
	function all_conn(connections, ids) {
		for (let [comp, conn] of connections) {
			var prev = conn.get("prev")
			var next = conn.get("next")
			if (prev.length == 0) {
				if (ids.get(comp)!="Strategy") {
					return false;
				}
			}
			if (next.length==0) {
				if (ids.get(comp)!="Distal outcome" && ids.get(comp)!="Proximal outcome" && ids.get(comp) !="Barrier") {
					return false;
				}
			}

		}
		return true;
	}
	

	function single_conn(connections, ids) {
		for (let [comp, conn] of connections) {
			if (conn.get("next").length > 1) {
				feedback="Your " + ids.get(comp) + " points to multiple components." +" Please make sure that you have only selected one CPD pathway, and that your Strategy points to only one component."
				return false;
			}
		}
		return true;
	}
 
	function process_content(content) {
		return content.replace(/<a\b[^>]*>/i,"").replace(/<\/a>/i, "").replace("<br /><br />"," :")
	}
	function set_component(comp_list, key, val) {
		var temp = comp_list.get(key)
		temp.push(val)
		comp_list.set(key, temp)
	}

	function req_comp_check(components) {
		if (components.get("Strategy").length == 0) {
			feedback_arr.push("Strategy")
		}
		if (components.get("Mechanism").length == 0) {
			feedback_arr.push("Mechanism")
		}
		if (components.get("Proximal outcome").length==0 && components.get("Distal outcome").length==0 && components.get("Barrier").length==0) {
			feedback_arr.push("Barrier")
			feedback_arr.push("An outcome: Proximal or Distal")
		}

		if (feedback_arr.length == 0){
			return true;
		}
		else{
			return false;
		}
	}
	function req_comp_extra(components) {
		components.forEach((val, key) => {
			if (val.length > 1) {
				feedback_arr.push(key)
			}
		});
		
		if (feedback_arr.length == 0){
			return true;
		}
		else{
			return false;
		}
	}


	function is_strategy(shape) {
		return shape=="round_rectangle"
	}
	function is_mechanism(shape) {
		return shape=="rhombus"
	}
	function is_barrier(shape) {
		return shape=="octagon"
	}
	function is_proximal(shape, content) {
		return shape=="circle" && process_content(content).includes("Proximal")
	}
	function is_distal(shape, content) {
		return shape=="circle" && process_content(content).includes("Distal")
	}
	function is_moderator(shape) {
		return shape=="rectangle"
	}
	function is_precondition(shape) {
		return shape=="trapezoid"
 }
	function process_response(recommended) {
		recommended.split("\n").filter(element => element.includes("- "));
	}

	// Based on selection about which component needs to be updated, update list
	function update_recommend() {
		recommending = false;
		if (default_comp=="Strategy") {
			prev_list=["NA"]
			next_list.splice(1, 1)
			
		}
		if (default_comp=="Barrier") {
			prev_list.splice(3, 1)
			next_list.splice(1, 1)
			next_list.splice(2, 1)
		}
		if (default_comp=="Proximal outcome") {
			prev_list.splice(5, 1)
		}
		if (default_comp=="Distal outcome") {
			next_list=["NA"]
		}
		


	}
	async function faq() {
		faq_sel=false;
		faq_feedback=[]
		faq_def=[]
		const items = await miro.board.getSelection();
		
		if (items.length == 0) {
			faq_feedback.push("Please select a component.")
			faq_sel=true
			return 
		}
		if (items.length > 1) {
			faq_feedback.push("Please select one and only one component.")
			faq_sel=true
			return
		}
		items.forEach((item) => {
			if (item.type != "connector") {
				if (is_strategy(item.shape)) {
					faq_feedback.push("The selected is a Strategy: \n")
					faq_def.push(faq_definitions[0])
				}
				if (is_mechanism(item.shape)) {
					faq_feedback.push("The selected is a Mechanism: \n")
					faq_def.push(faq_definitions[1])
				}
				if (is_barrier(item.shape)) {
					faq_feedback.push("The selected is a Barrier: \n")
					faq_def.push(faq_definitions[2])
				}
				if (is_proximal(item.shape, item.content)) {
					faq_feedback.push("The selected is a Proximal outcome: \n")
					faq_def.push(faq_definitions[3])
				}
				if (is_distal(item.shape, item.content)) {
					faq_feedback.push("The selected is a Distal outcome: \n")
					faq_def.push(faq_definitions[4])
				}
				if (is_precondition(item.shape)) {
					faq_feedback.push("The selected is a Precondition: \n")
					faq_def.push(faq_definitions[5])
				}
				if (is_moderator(item.shape)) {
					faq_feedback.push("The selected component is a Moderator: \n")
					faq_def.push(faq_definitions[6])
				}
			} 
		});
		faq_sel=true
	}
	function restart_brainstorming() {
		recommending=true
		component_names = [
		"Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"
		];
	   default_comp = "Strategy"

		prev_list = ["NA","Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"]
		next_list= ["NA","Strategy",
		"Mechanism",
		"Barrier",
		"Proximal outcome",
		"Distal outcome"]
		prev_comp="NA";
		prev_input="";
		next_comp="NA";
		next_input="";
		return
	}
	onMount(
		async () => {
		async function moderator(x, y) {
			const shape = await miro.board.createShape({
				shape: "rectangle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.ug2co4bl0tku">Moderator</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});
		}
		async function moderator_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "rectangle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.ug2co4bl0tku">Moderator</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});
			return shape
			
		}

		async function strategy(x, y) {
			const shape = await miro.board.createShape({
				shape: "round_rectangle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.od8pcqdtkhl2">Strategy</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});

			const shape_end = await miro.board.createShape({
				shape: "circle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 12, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "transparent", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x + 200, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 10,
				height: 10,
			});

			const connector = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: shape_end.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
		}
		async function strategy_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "round_rectangle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.od8pcqdtkhl2">Strategy</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});
			return shape.id
		}
		async function mechanism(x, y) {
			const shape = await miro.board.createShape({
				shape: "rhombus",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.vqklszz7pj1i">Mechanism</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y + 20, // Default value: center of the board
				width: 160,
				height: 130,
			});

			const shape_end = await miro.board.createShape({
				shape: "circle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 12, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "transparent", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x + 200, // Default value: center of the board
				y: y + 20, // Default value: center of the board
				width: 10,
				height: 10,
			});

			const connector = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: shape_end.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
		}
		async function mechanism_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "rhombus",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.vqklszz7pj1i">Mechanism</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y + 20, // Default value: center of the board
				width: 150,
				height: 120,
			});

			return shape.id
		}

		async function precondition(x, y) {
			const shape = await miro.board.createShape({
				shape: "trapezoid",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.ld4fb457u03l">Precondition</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});
		}
		async function precondition_recom(x, y) {
			const shape = await miro.board.createShape({
				shape: "trapezoid",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.ld4fb457u03l">Precondition</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});
			return shape
		}
		

		async function barrier(x, y) {
			const shape = await miro.board.createShape({
				shape: "octagon",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.wf7exwrr03uh">Barrier</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y + 18.5, // Default value: center of the board
				width: 120,
				height: 120,
			});

			const shape_end = await miro.board.createShape({
				shape: "circle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 12, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "transparent", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x + 200, // Default value: center of the board
				y: y + 18.5, // Default value: center of the board
				width: 10,
				height: 10,
			});

			const connector = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: shape_end.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
		}
		async function barrier_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "octagon",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.wf7exwrr03uh">Barrier</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y + 20, // Default value: center of the board
				width: 120,
				height: 120,
			});
			return shape.id
		}

		async function proximal_outcome(x, y) {
			const shape = await miro.board.createShape({
				shape: "circle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.zbc933au5vfi">Proximal outcome</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 120,
				height: 120,
			});

			const shape_end = await miro.board.createShape({
				shape: "circle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 12, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "transparent", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x + 200, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 10,
				height: 10,
			});

			const connector = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: shape_end.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
		}
		async function proximal_outcome_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "circle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.zbc933au5vfi">Proximal outcome</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 140,
				height: 140,
			});
			return shape.id
		}

		async function distal_outcome(x, y) {
			const shape = await miro.board.createShape({
				shape: "circle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.o3ahs47uleu4">Distal outcome</a>',
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 140,
				height: 140,
			});
			return shape
		}
		async function distal_outcome_recom(x, y, content) {
			const shape = await miro.board.createShape({
				shape: "circle",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.o3ahs47uleu4">Distal outcome</a><br><br>'+content,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 120,
				height: 120,
			});
			return shape.id
		}
		async function connector_recom(start, end) {
			const connector = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: start,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: end,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
			return connector
		}

		async function flow(x, y, strategy_flow, mechanism_flow, barrier_flow, prox_out_flow, dist_out_flow, mod_flow, prec_flow) {
			const strategy_shape = await miro.board.createShape({
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.od8pcqdtkhl2">Strategy</a><br><br>' +
					strategy_flow,
				shape: "round_rectangle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x - 600, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 150,
				height: 75,
			});

			const mechanism_shape = await miro.board.createShape({
				shape: "rhombus",
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.vqklszz7pj1i">Mechanism</a><br><br>' +
					mechanism_flow,
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x - 300, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 175,
				height: 135,
			});

			const barrier_shape = await miro.board.createShape({
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.wf7exwrr03uh">Barrier</a><br><br>' +
					barrier_flow,
				shape: "octagon",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 120,
				height: 120,
			});


			const prox_out_shape = await miro.board.createShape({
							content:
								'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.zbc933au5vfi">Proximal outcome</a><br><br>' +
								prox_out_flow,
							shape: "circle",
							style: {
								color: "#000", // Default text color: '#1a1a1a' (black)
								fillColor: "transparent", // Default shape fill color: transparent (no fill)
								fontFamily: "arial", // Default font type for the text
								fontSize: 11, // Default font size for the text, in dp
								textAlign: "center", // Default horizontal alignment for the text
								textAlignVertical: "middle", // Default vertical alignment for the text
								borderStyle: "normal", // Default border line style
								borderOpacity: 1.0, // Default border color opacity: no opacity
								borderColor: "#000", // Default border color: '#ffffff` (white)
								borderWidth: 1, // Default border width
								fillOpacity: 1.0, // Default fill color opacity: no opacity
							},
							x: x + 300, // Default value: center of the board
							y: y, // Default value: center of the board
							width: 120,
							height: 120,
						});

			const dist_out_shape = await miro.board.createShape({
				content:
					'<a href="https://docs.google.com/document/d/1GhgTo2pnpPTqXW9YbmSdKMT96rK6G7VpX8JmOq5Mod0/edit#bookmark=id.o3ahs47uleu4">Distal outcome</a><br><br>' +
					dist_out_flow,
				shape: "circle",
				style: {
					color: "#000", // Default text color: '#1a1a1a' (black)
					fillColor: "transparent", // Default shape fill color: transparent (no fill)
					fontFamily: "arial", // Default font type for the text
					fontSize: 11, // Default font size for the text, in dp
					textAlign: "center", // Default horizontal alignment for the text
					textAlignVertical: "middle", // Default vertical alignment for the text
					borderStyle: "normal", // Default border line style
					borderOpacity: 1.0, // Default border color opacity: no opacity
					borderColor: "#000", // Default border color: '#ffffff` (white)
					borderWidth: 1, // Default border width
					fillOpacity: 1.0, // Default fill color opacity: no opacity
				},
				x: x + 600, // Default value: center of the board
				y: y, // Default value: center of the board
				width: 120,
				height: 120,
			});

			const connector_first = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: strategy_shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: mechanism_shape.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});

			const connector_second = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: mechanism_shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: barrier_shape.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});

			const connector_third = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: barrier_shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: prox_out_shape.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});


			const connector_fourth = await miro.board.createConnector({
				shape: "elbowed",
				style: {
					endStrokeCap: "rounded_stealth",
					strokeStyle: "normal",
					strokeColor: "#000", // Magenta
					strokeWidth: 2,
				},
				start: {
					item: prox_out_shape.id,
					position: {
						x: 1.0,
						y: 0.5,
					},
				},
				end: {
					item: dist_out_shape.id,
					position: {
						x: 0.0,
						y: 0.5,
					},
				},
			});
		}

		async function recommend_draw(x, y, prev, curr, next) {
			if (prev == "NA" && next == "NA") {
				return
			}	
			// if (prev != "NA" && prev_input=="") {
			// 	return
			// }
			// if (next != "NA" && next_input=="") {
			// 	return
			// }
			initialize_definitions()
			var recommend_instruction = "The user is building a causal pathway diagram (CPD), each component of which is defined as follows:\n\n"
			var temp = 0
			for (var i=1; i<temp + 1; i++) {
				recommend_instruction += "- " + entities[i] + ": " + questions_instruction[i] + "\n"
			}
			// // define user message
			let user_message = ""
			if (prev == "NA") {
				recommend_instruction += '\nBased on the '+next+ 'the user has input, the system should recommend 5 probable '+ curr + '. Each response should start with "- ", and each response should be very short. The output should not end with "."';
				user_message += next+": " + next_input+"\n"
			} else if (next == "NA") {
				recommend_instruction += '\nBased on the '+prev+ 'the user has input, the system should recommend 5 probable '+ curr + '. Each response should start with "- ", and each response should be very short. The output should not end with "."';
				user_message += prev+": " + prev_input+"\n"
			} else {
				recommend_instruction += '\nBased on the '+prev+ ' and ' + next+ ' the user has input, the system should recommend 5 probable '+ curr + '. Each response should start with "- ", and each response should be very short. The output should not end with "."';
				user_message += prev+": " + prev_input+"\n"
				user_message += next+": " + next_input+"\n"
			}
			

			const recommended = await getRecommendations(recommend_instruction, user_message).then((response) => {
					arr = response.split("\n").filter(element => element.includes("- "));
					for (var i = 0; i < arr.length; i++) {
						arr[i] = arr[i].replace("- ", "").trim();
						arr = arr;
					}
					
			});
			
			let yindex=[300, 150, 0, -150, -300]
			if (prev=="NA" || prev =="") {
				var end;
				if (next=="Strategy") {
					end = await strategy_recom(x, y, next_input)				
				} else if (next=="Mechanism") {
					end = await mechanism_recom(x, y, next_input)
				} else if (next=="Barrier") {
					end = await barrier_recom(x, y, next_input)
				} else if (next=="Proximal outcome") {
					end = await proximal_outcome_recom(x, y, next_input)
				} else if (next=="Distal outcome") {
					end = await distal_outcome_recom(x, y, next_input)
				}
				for (var i = 0; i < arr.length; i++) {
						if (curr=="Strategy") {
							var start = await strategy_recom(x-300, y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Mechanism") {
							var start = await mechanism_recom(x-300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Barrier") {
							var start = await barrier_recom(x-300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Proximal outcome") {
							var start = await proximal_outcome_recom(x-300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Distal outcome") {
							var start = await distal_outcome_recom(x-300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						}	
					
				}
			} else if (next == "NA" || next =="") {
				var start;
				if (prev=="Strategy") {
					start = await strategy_recom(x, y, prev_input)				
				} else if (prev=="Mechanism") {
					start = await mechanism_recom(x, y, prev_input)
				} else if (prev=="Barrier") {
					start = await barrier_recom(x, y, prev_input)
				} else if (prev=="Proximal outcome") {
					start = await proximal_outcome_recom(x, y, prev_input)
				} else if (prev=="Distal outcome") {
					start = await distal_outcome_recom(x, y, prev_input)
				}
				
				for (var i = 0; i < arr.length; i++) {
						if (curr=="Strategy") {
							var end = await strategy_recom(x+300, y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Mechanism") {
							var end = await mechanism_recom(x+300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Barrier") {
							var end = await barrier_recom(x+300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Proximal outcome") {
							var end =await proximal_outcome_recom(x+300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						} else if (curr=="Distal outcome") {
							var end = await distal_outcome_recom(x+300,  y+yindex[i], arr[i])
							var conn = await connector_recom(start, end)
						}	
					
				}
			} else {
				
				var start;
				if (prev=="Strategy") {
					start = await strategy_recom(x, y, prev_input)				
				} else if (prev=="Mechanism") {
					start = await mechanism_recom(x, y, prev_input)
				} else if (prev=="Barrier") {
					start = await barrier_recom(x, y, prev_input)
				} else if (prev=="Proximal outcome") {
					start = await proximal_outcome_recom(x, y, prev_input)
				} else if (prev=="Distal outcome") {
					start = await distal_outcome_recom(x, y, prev_input)
				}
				var end;
				if (next=="Strategy") {
					end = await strategy_recom(x+600, y, next_input)				
				} else if (next=="Mechanism") {
					end =await  mechanism_recom(x+600, y, next_input)
				} else if (next=="Barrier") {
					end = await barrier_recom(x+600, y, next_input)
				} else if (next=="Proximal outcome") {
					end = await proximal_outcome_recom(x+600, y, next_input)
				} else if (next=="Distal outcome") {
					end = await distal_outcome_recom(x+600, y, next_input)
				}
				for (var i = 0; i < arr.length; i++) {
						if (curr=="Strategy") {
							var mid = await strategy_recom(x+300, y+yindex[i], arr[i])
							var conn1 = await connector_recom(start, mid)
							var conn2 = await connector_recom(mid, end)
						} else if (curr=="Mechanism") {
							var mid = await mechanism_recom(x+300,  y+yindex[i], arr[i])
							var conn1 = await connector_recom(start, mid)
							var conn2 = await connector_recom(mid, end)
						} else if (curr=="Barrier") {
							var mid = await barrier_recom(x+300,  y+yindex[i], arr[i])
							var conn1 = await connector_recom(start, mid)
							var conn2 = await connector_recom(mid, end)
						} else if (curr=="Proximal outcome") {
							var mid = await proximal_outcome_recom(x+300,  y+yindex[i], arr[i])
							var conn1 = await connector_recom(start, mid)
							var conn2 = await connector_recom(mid, end)
						} else if (curr=="Distal outcome") {
							var mid = await distal_outcome_recom(x+300,  y+yindex[i], arr[i])
							var conn1 = await connector_recom(start, mid)
							var conn2 = await connector_recom(mid, end)
						}	
					
				}
			}
				
		
			
	}

		async function drop() {
			await miro.board.ui.on("drop", async ({ x, y, target }) => {
				switch (target.id) {
					case "1":
						moderator(x, y);
						break;
					case "2":
						strategy(x, y);
						break;
					case "3":
						precondition(x, y);
						break;
					case "4":
						mechanism(x, y);
						break;
					case "5":
						barrier(x, y);
						break;
					case "6":
						proximal_outcome(x, y);
						break;
					case "7":
						distal_outcome(x, y);
						break;
					case "11":
						flow(x, y, responses[3], responses[4], responses[1], responses[2], responses[0], moderator_input, precondition_input);
						break;
					case "15":
						recommend_draw(x, y, prev_comp, default_comp, next_comp);
						break;
					default:
						break;
				}
			});
		}

		drop();

		parseScroll();
	});
	
	let active = "Components";
</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Svelte demo app" />
	<script src="https://miro.com/app/static/sdk/v2/miro.js"></script>
</svelte:head>

<section>
	<TabBar tabs={["Components", "Wizard", "Brainstorming","Checking", "Help"]} let:tab bind:active>
		<Tab {tab}>
			<Label>{tab}</Label>
		</Tab>
	</TabBar>
	{#if active == "Components"}
		<div class="form">
			<div class="container m-0 py-4 bg-transparent">
				<div class="row m-0 p-0">
					<div
						class="col-6 miro-draggable"
						id="1"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="A factor that strengthens or weakens the relationship between the strategy and mechanism"
					>
						<p class="heading">Moderator</p>
						<svg
							width="100%"
							height="80"
							style="text-align: center;"
						>
							<rect
								x="15%"
								y="5"
								width="70%"
								height="50"
								stroke="black"
								fill="transparent"
								class="rect"
							/>
						</svg>
					</div>

					<div
						class="col-6 miro-draggable"
						id="2"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="A specific and operationally defined implementation strategy"
					>
						<p class="heading">Implementation strategy</p>
						<svg width="100%" height="70">
							<rect
								x="15%"
								y="5"
								width="70%"
								height="50"
								rx="10"
								ry="10"
								class="rect"
								stroke="black"
								fill="transparent"
							/>
						</svg>
					</div>
					<div
						class="col-6 miro-draggable"
						id="3"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="A factor that is necessary for the mechanism to be activated and subsequent causal events to occur"
					>
						<p class="heading">Precondition</p>
						<svg
							version="1.0"
							xmlns="http://www.w3.org/2000/svg"
							width="90%"
							height="70"
							style="margin-left:5%"
							viewBox="0 0 300.000000 131.000000"
							preserveAspectRatio="xMidYMid "
						>
							<g
								transform="translate(0.000000,131.000000) scale(0.100000,-0.100000)"
								fill="#1D1D1D"
								stroke="none"
							>
								<path
									d="M699 1218 c-51 -89 -460 -793 -539 -928 -133 -226 -159 -275 -150
					-284 8 -8 2990 2 2990 11 0 2 -142 250 -316 551 -174 301 -342 591 -372 645
					l-57 97 -751 0 -752 0 -53 -92z m1549 50 c5 -7 45 -74 88 -148 42 -74 109
					-189 147 -255 38 -66 138 -239 222 -385 84 -146 175 -303 203 -350 28 -47 47
					-88 43 -93 -4 -4 -660 -6 -1459 -5 l-1451 3 180 310 c99 171 195 335 213 365
					18 30 100 171 182 312 l149 258 736 0 c577 0 739 -3 747 -12z"
									id="node1"
									stroke-width="15"
								/>
							</g>
							<g
								transform="translate(0.000000,131.000000) scale(0.100000,-0.100000)"
								fill="#9D9D9D"
								stroke="none"
							/>
							<g
								transform="translate(0.000000,131.000000) scale(0.100000,-0.100000)"
								fill="#FFFFFF"
								stroke="none"
							>
								<path
									d="M757 1258 c-8 -13 -73 -124 -144 -248 -72 -124 -226 -389 -343 -588
					-116 -200 -214 -371 -217 -379 -5 -14 143 -15 1443 -8 797 4 1451 9 1453 11 2
					2 -10 25 -27 51 -16 26 -88 149 -159 273 -71 124 -218 379 -327 568 l-198 342
					-734 0 -734 0 -13 -22z"
									id="node9"
									stroke-width="15"
								/>
							</g>
						</svg>
					</div>

					<div
						class="col-6 miro-draggable"
						id="4"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="How or why the strategy works to resolve the barrier"
					>
						<p class="heading">Mechanism</p>
						<svg
							width="100%"
							height="90"
							transform="rotate(-45 -10, -60)"
							stroke="black"
							fill="transparent"
						>
							<rect
								x="5"
								y="5"
								width="50"
								height="50"
								class="rect"
								stroke="black"
								fill="transparent"
							/>
						</svg>
					</div>

					<div
						class="col-6 miro-draggable"
						id="5"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="The problem that a strategy is intended to resolve"
					>
						<p class="heading">Barrier</p>
						<svg viewBox="-100 4 400 300">
							<polygon xmlns="http://www.w3.org/2000/svg" style="fill:none;stroke:#000000;stroke-width:2.5px" points="136.737609507049,188.692435121084 63.2623904929514,188.692435121084 11.3075648789165,136.737609507049 11.3075648789165,63.2623904929514 63.2623904929513,11.3075648789165 136.737609507049,11.3075648789165 188.692435121084,63.2623904929513 188.692435121084,136.737609507049"/>
						</svg>
					</div>

					<div
						class="col-6 miro-draggable"
						id="6"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="The earliest signs of change in the mechanism, barrier, or precursors to the distal outcome"
					>
						<p class="heading">Proximal outcome</p>
						<svg width="100%" height="80">
							<circle
								cx="50%"
								cy="40"
								r="33"
								class="rect"
								stroke="black"
								fill="transparent"
							/>
						</svg>
					</div>

					<div
						class="col-6 miro-draggable"
						id="7"
						data-bs-toggle="tooltip"
						data-bs-placement="top"
						title="The desired implementation outcome"
					>
						<p class="heading">Distal outcome</p>
						<svg width="100%" height="80">
							<circle
								cx="50%"
								cy="40"
								r="33"
								class="rect"
								stroke="black"
								fill="transparent"
							/>
						</svg>
					</div>
				</div>
			</div>
		</div>
	{:else if active == "Checking"}
		<div class="form">
		<div class="paper-container">
			<div class="mdc-typography--body2" style="font-size:0.9rem"><i><small><strong>You can use this feature to explore if a CPD follows the basic rules. Select a CPD pathway on the board to proceed.</strong></small></i></div>
			<br>
			<br>
			<div class="mdc-typography--headline1"><big>Select a CPD pathway</big></div>
			<div style = "display: flex; justify-content:flex-end; padding: 20px">
				<Button on:click={checkCPD} variant="raised">
					<Label> Check </Label>
				</Button>
			</div>
			{#if not_all_req}
				<Paper variant="outlined" style="padding:20px">
					<!-- <pre class="status"></pre> -->
						<Content>
							<p class="checking_header" style="font-weight:bold">{feedback}</p>
							<br>
							<ul>
								{#each feedback_arr as f}
									<li class="header">{f}</li>
								{/each}
							</ul>
						</Content>
					<!-- </pre> -->

				</Paper>
			{/if}
			{#if comp_extra}
				<Paper variant="outlined" style="padding:20px">
					<Content>
							<p class="checking_header" style="font-weight:bold">{feedback}</p>
							<br>
							<ul>
								{#each feedback_arr as f}
									<li class="header">{f}</li>
								{/each}
							</ul>
						
					</Content>

				</Paper>
			{/if}
			{#if multiple_conn || miss_conn || wrong_coord}
			<Paper variant="outlined" style="padding:20px">
				<Content>
						<p class="checking_header" style="font-weight:bold">{feedback}</p>
					
				</Content>

			</Paper>
		{/if}
		{#if not_corr_conn ||basic_conn}
				<Paper variant="outlined" style="padding:20px">
					<!-- <pre class="status"></pre> -->
						<Content>
							<p class="checking_header" style="font-weight:bold">{feedback}</p>
							<br>
							<ul>
								{#each feedback_arr as f}
									<li class="header">{f}</li>
								{/each}
							</ul>
						</Content>
					<!-- </pre> -->

				</Paper>
			{/if}
		</div>
	</div>
	{:else if active =="Brainstorming"} 
		{#if recommending}
		<div class="form">
			<div class="mdc-typography--body2" style="font-size:0.9rem"><i><small><strong>You can use this feature to help brainstorm the content for a specific component. First, select the component you want to brainstorm. Then on the next screen, you can input the components that comes before and/or after it. </strong></small></i></div>
			<br>
			<br>
			<div class="header"><p>Which component would you like to brainstorm?</p></div>
			
			
			<!-- <div style = "display: flex; justify-content:flex-end; padding: 20px"> -->
			<Select bind:value={default_comp}>
				{#each component_names as comp}
					<Option value={comp}>{comp}</Option>
				{/each}
				</Select>

			<!-- </div> -->
			<br>
			<div style = "display: flex; justify-content:flex-end; padding: 20px">
				<Button on:click={update_recommend} variant="raised">
					<Label> Next > </Label>
				</Button>
				</div>

			<!-- <Button id="15" class="miro-draggable" variant="raised">
				<Label> Brainstorm </Label>
			</Button> -->
		</div>
		{:else}
			<!-- <div class ="paper-container"> -->
				<br>
				
			<!-- </div> -->
				
				<div class="form">
				
				<div class="header"><p>What is the component before the brainstorming component ({default_comp})?</p></div>
				<Select bind:value={prev_comp}>
					{#each prev_list as comp}
						<Option value={comp}>{comp}</Option>
					{/each}
					</Select>
				<!-- </div> -->
				{#if prev_comp!="NA"}
					<Textfield class="input_field" variant="filled" label={prev_comp} bind:value={prev_input} />
				{/if}
				<br>
				<br>
				<br>
				<div class="header"><p>What is the component after the brainstorming component ({default_comp})?</p></div>
				<Select bind:value={next_comp}>
					{#each next_list as comp}
						<Option value={comp}>{comp}</Option>
					{/each}
					</Select>
				{#if next_comp != "NA"}
				<Textfield class="input_field" variant="filled" label={next_comp} bind:value={next_input} />
				{/if}
				<br>
				<div style = "display: flex; justify-content:flex-end; padding: 20px">
					<Button on:click={restart_brainstorming} variant="raised" style="margin-top: 10px !important;margin-right: 10px">
						<Label> Restart </Label>
					</Button>

				<Button id="15" class="miro-draggable" variant="raised" style="margin-left: 30px;margin-top: 10px !important">
						<Label> Recommend    (Drag & Drop) </Label>
					</Button>
					
				</div>
				</div>

		{/if}
		
	
	{:else if active == "Help"}
		<div class="form">
		<div class="paper-container">
			<div class="mdc-typography--body2" style="font-size:0.9rem"><i><small><strong>You can use this feature to explore the details of each component. Select any component on the board to learn about what it is and how it is used in a CPD.</strong></small></i></div>
			<br>
			<br>
			<div class="mdc-typography--headline1"><big>Select a component</big></div>
			<!-- <Title></Title> -->
			<div style = "display: flex; justify-content:flex-end; padding: 20px">
				
				<Button on:click={faq} variant="raised">
					<Label> Learn More </Label>
				</Button>
			</div>
			{#if faq_sel}
	
		<Paper variant="outlined">
			<Content>
				{#each faq_feedback as f, index}
					<p class="checking_header" style="font-weight:bold">{f}</p>
					<br>
					{#if faq_def.length > 0}
					<p>{faq_def[index]}</p>
					<br>
					{/if}
					{/each}
			</Content>
		
			
		</Paper>
		{/if}
		</div>
		
		</div>
		
		

	{:else}
	<div class="form">
		{#if inProgress}
		<div style="display: flex; justify-content: center; margin-top: 1rem">
			<CircularProgress style="height: 32px; width: 32px;" indeterminate />
		</div>
		<br>
		<p class="heading">Generating recommendations..</p>
		{:else}
			{#if index == 0}
			<div class="mdc-typography--body2" style="font-size:0.9rem"><i><small><strong>This feature will guide you through the process of generating a CPD.</strong></small></i></div>
			
				<br>
			{/if}
		<Label class="header">{questions[index]}</Label>
		<Textfield class="input_field" variant="filled" label={entities[index]} bind:value={responses[index]} />
		{#if response_array.length != 0}
		<div style="margin-top: 1rem;">
			<p class="header" style="margin-bottom: 10px !important">AI-generated recommendations</p>
			{#each response_array as res}
			<Button style="text-transform: none !important; margin: 0.25rem !important; padding: 0 15px !important; color: #333 !important; background-color: #ededed !important; border-radius: 20px; text-align: left; font-size: .75rem !important; font-weight: 500 !important;" on:click={() => {responses[index] = res; responses=responses;}}>
				{res}
			</Button>
			{/each}
		</div>
		{/if}
		<div style = "display: flex; justify-content:flex-end; padding: 20px">
			<!-- <div style = "display: flex; justify-content:flex-end; padding: 20px"> -->
				<!-- style="text-align: right !important;" -->
			{#if index==4}
				<Button on:click={restart_wizard} variant="raised" style="margin-top: 10px !important;margin-right: 10px">
					<Label> Restart </Label>
				</Button>
			{/if}
			
			<Button id="11" variant="raised" style="margin-top: 10px !important; margin-left:30px" class="{index == 4 ? 'miro-draggable':''}" on:click={() => {
				if (index != 4) {
					inProgress = true;
					// define instruction
					let instruction = "The user is building a causal pathway diagram (CPD), each component of which is defined as follows:\n\n"
					for (var i=0; i<5; i++) {
						instruction += "- " + entities[i] + ": " + questions_instruction[i] + "\n"
					}
					instruction += `\nBased on the ${entities.slice(0, index+1).map((str) => str.toLowerCase()).join(', ')} the user has input, the system should recommend 5 probable ${Pluralize(entities[index+1], 10, false).toLowerCase()}. Each response should start with "- ", and each response should be very short. The output should not end with "."`;


					// define user message
					let user_message = ""
					for (var i=0; i<index+1; i++) {
						user_message += entities[i] + ": " + responses[i] + "\n"
					}
					console.log(instruction, user_message);
					getRecommendations(instruction, user_message).then((response) => {
						response_array = response.split("\n").filter(element => element.includes("- "));
						for (var i = 0; i < response_array.length; i++) {
							response_array[i] = response_array[i].replace("- ", "").trim();
							response_array = response_array;
						}
						index += 1;
						inProgress = false;
					});
				}
			}
			} disabled={responses[index].trim().length == 0}>{#if index != 4}<label>Next ></label>{:else}<label>Generate CPD (Drag & Drop)</label>{/if}</Button>
		</div>
		{/if}
	</div>
	{/if}
</section>










<style>
	@import url('https://fonts.googleapis.com/css2?family=Inter:wght@100;200;300;400;500;600;700;800;900&display=swap');
	body {
		font-family: 'Inter', sans-serif;
	}
	.title {
		font-size: 1.2rem;
		font-weight: 600;
	}
	.subtitle {
		font-size: 0.9rem;
		font-weight: 300;
		padding-top: 10px;
	}
	.heading {
		text-align: center;
		font-weight: 400;
		font-size: 0.8rem;
		color: #333;
		padding-bottom: 5px;
	}
	:global(.header) {
		color: #666;
		font-weight: 400 !important;
		font-size: 0.9rem;
	}
	:global(.input_field) {
		width: 100% !important;
		margin: 10px 0 20px 0 !important;
	}
	.heading_unit {
		font-weight: 300;
		font-size: 90%;
		float: right;
		padding-top: 3px;
	}
	.form {
		padding: 15px;
		margin-bottom: 1rem;
		height: 90vh;
		overflow-y: scroll !important;
	}
	:global(.mdc-tab__text-label) {
		font-family: 'Inter', sans-serif;
		text-transform: none;
	}
	.unit-main {
		margin-bottom: 2rem;
	}

	h1 {
		width: 100%;
	}
	.paper-container {
		/* padding:20px; */
		margin-bottom: 1rem;
		overflow-y: scroll !important;
	}
	:global(.checking_header) {
		color: #666;
		font-weight: bold;
		font-size: 1rem;
	}
</style>
