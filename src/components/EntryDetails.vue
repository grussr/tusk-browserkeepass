<template>
	<div>
		<go-back :message="'back to entry list'"></go-back>
		<div class="all-attributes">

			<div v-if="otp" class="attribute-box">
				<span class="attribute-title">One Time Password</span>
				<br>
				<span class="attribute-value">{{otp_value}}</span>
				<div class="progress">
			      <div class="determinate" v-bind:style="{ width: otp_width }"></div>
			  </div>
			</div>
			
			<div class="attribute-box" v-for="attr in attributes">
				<span class="attribute-title">{{ attr.key }}</span>
				<br>
				<pre v-if="attr.key == 'notes'" class="attribute-value">{{ attr.value }}</pre>
				<span v-else-if="!attr.protected" class="attribute-value">{{ attr.value }}</span>
				<div v-else>
					<span v-if="attr.key !== 'notes'" class="attribute-value protected" @click="toggleAttribute(attr)">
            <i v-if="attr.protected && attr.isHidden" 
              class="fa fa-eye-slash" aria-hidden="true"></i>
            <i v-else-if="attr.protected && !attr.isHidden"
              class="fa fa-eye" aria-hidden="true"></i>
            {{ attr.value }}
          </span>
				</div>
			</div>
		
		</div>
	</div>
</template>

<script>
	const OTP = require('keeweb/app/scripts/util/otp.js')
	import GoBack from '@/components/GoBack'

	export default {
		components: {
			GoBack
		},
		props: {
			unlockedState: Object,
			settings: Object
		},
		data() {
			return {
				attributes: [],
				hiddenValue: '••••••••••',
				// OTP
				otp: false,
				otp_timeleft: 0,
				otp_loop: undefined,
				otp_value: "",
				otp_width: 0
			}
		},
		methods: {
			exposeAttribute(attr) {
				attr.value = this.unlockedState.getDecryptedAttribute(this.entry, attr.key)
				attr.isHidden = false
			},
			hideAttribute(attr) {
				attr.value = this.hiddenValue;
				attr.isHidden = true
			},
			toggleAttribute(attr) {
				if (attr.isHidden)
					this.exposeAttribute(attr)
				else
					this.hideAttribute(attr)
			},
			setupOTP(url) {
				let otpobj = OTP.parseUrl(url)
				this.otp = true
				let do_otp = () => {
					otpobj.next((code, timeleft)=>{
						this.otp_value = code;
						this.otp_timeleft = (timeleft / 1000) | 0;
						this.otp_width = Math.floor(timeleft / 300) + "%"
					})
				}
				this.otp_loop = setInterval(do_otp, 2000)
				do_otp()
			}
		},
		beforeDestroy(){
			clearInterval(this.otp_loop)
		},
		mounted() {
			let entryId = this.$router.getRoute().entryId
			this.entry = this.unlockedState.cacheGet('allEntries').filter(entry => {
				return entry.id == entryId
			})[0]
			this.attributes = this.entry.keys.map(key => {
				// Should NOT be succeptible to XSS
				let value = key !== 'notes' ?
					(this.entry[key] || "").replace(/\n/g, "<br>") :
					this.entry[key]
				return {
					'key': key,
					'value': value
				}
			})
			for (var protectedKey in this.entry.protectedData) {
				if (protectedKey === "otp") {
					let url = this.unlockedState.getDecryptedAttribute(this.entry, protectedKey)
					this.setupOTP(url)
				} else {
					this.attributes.push({
						'key': protectedKey,
						'value': this.hiddenValue,
						'isHidden': true,
						'protected': true,
						'protectedAttr': this.entry.protectedData[protectedKey]
					})
				}
			}
		}
	}
</script>

<style lang="scss">
	@import "../styles/settings.scss";
	.all-attributes {
		max-height: 400px;
		overflow-y: auto;
	}

	.attribute-box {
		box-sizing: border-box;
		padding: 8px $wall-padding;
		font-size: 16px;
		background-color: $light-background-color;
	}

	.attribute-title {
		padding-bottom: 10px;
		font-weight: 700;
		font-size: 12px;
	}

	.attribute-value {
		font-family: "DejaVu Sans", Arial, sans-serif;
		&.protected:hover {
			outline: $light-gray solid 2px;
			outline-offset: 1px;
		}
	}
</style>