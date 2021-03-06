<script lang="typescript">
  import NavBar from "../components/NavBar.svelte";
  import Footer from "../components/Footer.svelte";
  import Button from "../components/Button.svelte";
  import Loading from "../components/Loading.svelte";
  import Modal from "../components/Modal.svelte";
  import { loggedIn, address, keyfile } from "../stores/keyfileStore.ts";
  import { goto } from "@sapper/app";
  import { fade } from "svelte/transition";

  import { query } from "../api-client";
  import tokensQuery from "../queries/tokens.gql";
  import Arweave from "arweave";
  import Community from "community-js";
  import { pstContract, exchangeWallet } from "../utils/constants";

  if (process.browser && !$loggedIn) goto("/");

  let supportedTokens = getSupportedPSTs();
  let addTokenModalOpened: boolean = false;
  let newContractID: string;

  async function getSupportedPSTs(): Promise<
    { id: string; name: string; ticker: string }[]
  > {
    if (!process.browser) return [];

    let psts: { id: string; name: string; ticker: string }[] = [];

    const client = new Arweave({
      host: "arweave.dev",
      port: 443,
      protocol: "https",
      timeout: 20000,
    });

    let txIds = [];
    const _txIds = (
      await query({
        query: tokensQuery,
        variables: {
          owners: [exchangeWallet],
          contractSrc: pstContract,
        },
      })
    ).data.transactions.edges;
    _txIds.map(({ node }) => {
      txIds.push(node.id);
    });

    for (const id of txIds) {
      try {
        const contractId = (
          await client.transactions.getData(id, { decode: true, string: true })
        ).toString();
        const contractData = JSON.parse(
          (
            await client.transactions.getData(contractId, {
              decode: true,
              string: true,
            })
          ).toString()
        );
        psts.push({
          id: contractId,
          name: contractData["name"],
          ticker: contractData["ticker"],
        });
      } catch (error) {
        console.log(error);
      }
    }

    return psts;
  }

  async function addToken() {
    addTokenModalOpened = true;
  }

  async function confirmAdd(cancelClose: Function) {
    if (newContractID === "" || newContractID === undefined) cancelClose();
    else {
      const client = new Arweave({
        host: "arweave.dev",
        port: 443,
        protocol: "https",
        timeout: 20000,
      });
      let community = new Community(client, JSON.parse($keyfile));
      await community.setCommunityTx(pstContract);

      // Make sure user has balance of our PST
      try {
        let balance = await community.get({
          function: "unlockedBalance",
        });

        if (balance.balance > 0) {
          console.log("YOUR BAL IS GREATER THAN 0");
        } else {
          // Show an error because they need to have a balance

          addTokenModalOpened = false;
        }
      } catch (err) {
        console.log(err);
      }

      // Propose a vote to the DAO
      await community.proposeVote({
        type: "indicative",
        note: newContractID,
      });
      newContractID = "";
    }
  }

  function cancelAdd() {
    addTokenModalOpened = false;
    newContractID = "";
  }

  function roundCurrency(val: number | string): string {
    if (val === "?") return val;
    if (typeof val === "string") val = parseFloat(val);
    return val.toFixed(7);
  }
</script>

<!-- prettier-ignore -->
<style lang="sass">

  @import "../styles/general.sass"

  .tokens
    @include page

    @media screen and (max-width: 720px)
      padding-top: 2em

    .tokens-head
      display: flex
      justify-content: space-between
      margin-bottom: 2em

      @media screen and (max-width: 720px)
        display: block

      h1.title
        margin: 0

        @media screen and (max-width: 720px)
          font-size: 1.35em
          margin-bottom: .3em

    .tokens-content
      @media screen and (max-width: 720px)
        width: 100vw - $mobileSidePadding * 2

      a.token
        $sidePadding: 2.3em
        padding: 1em $sidePadding
        background-color: #161616
        text-decoration: none
        color: #fff
        display: flex
        align-items: center
        border-radius: 5px
        margin-bottom: 2.8em
        transition: all .3s

        @media screen and (max-width: 720px)
          padding: 1em 1.1em
          display: block

        &:last-child
          margin-bottom: 0

        &:hover
          background-color: #000000

        h1
          margin: 0
          
          &.short
            font-size: 3.1em
            color: #fff
            font-weight: 500
            text-transform: uppercase
            display: inline-block

            @media screen and (max-width: 1500px)
              font-size: 2.87em

            @media screen and (max-width: 720px)
              font-size: 2.1em

          &.val
            margin-left: auto
            font-size: 1.9em
            font-weight: 400

            @media screen and (max-width: 1500px)
              font-size: 1.67em

            span
              text-transform: uppercase
              font-size: .43em

        .info
          margin-left: 2em

          @media screen and (max-width: 720px)
            margin-left: 0

          h1
            font-size: 1.5em
            margin-bottom: .3em
            font-weight: 400

            @media screen and (max-width: 1500px)
              font-size: 1.27em

          p
            margin: 0
            font-size: 1.05em
            color: #fff

            @media screen and (max-width: 1500px)
              font-size: .82em

          span
            text-transform: uppercase
            color: #828282
            margin-right: .5em

  \:global(.Modal.cancelled input.contract-id) // opt out from scoped style
    border-color: #ff0000 // on modal close cancelled, show that the contract-id field was wrong
    animation: inputError .6s cubic-bezier(.36, .07, .19, .97) both
    transform: translate3d(0, 0, 0)
    backface-visibility: hidden

    @keyframes inputError
      10%, 90%
        transform: translate3d(-1px, 0, 0)
      
      20%, 80%
        transform: translate3d(2px, 0, 0)

      30%, 50%, 70%
        transform: translate3d(-4px, 0, 0)

      40%, 60%
        transform: translate3d(4px, 0, 0)

</style>

<svelte:head>
  <title>Verto — Tokens</title>
</svelte:head>

<NavBar />
<div class="tokens" in:fade={{ duration: 300 }}>
  <div class="tokens-head">
    <h1 class="title">Supported Tokens</h1>
    <Button
      click={addToken}
      style={"font-family: 'JetBrainsMono', monospace; text-transform: uppercase;"}>
      Add Token
    </Button>
  </div>
  <div class="tokens-content">
    {#await supportedTokens}
      <Loading />
    {:then loadedPSTs}
      {#each loadedPSTs as pst}
        <a
          class="token"
          href="https://viewblock.io/arweave/tx/{pst.id}"
          target="_blank">
          <h1 class="short">{pst.ticker}</h1>
          <div class="info">
            <h1><span>[PST]</span>{pst.name}</h1>
            <p><span>ID:</span>{pst.id}</p>
          </div>
        </a>
      {/each}
    {/await}
  </div>
</div>
<Modal
  bind:opened={addTokenModalOpened}
  confirmation={true}
  onConfirm={confirmAdd}
  onCancel={cancelAdd}>
  <h3 style="text-align: center;">Token Contract ID</h3>
  <input
    type="text"
    bind:value={newContractID}
    class="light contract-id"
    placeholder="Contract ID" />
</Modal>
<Footer />
