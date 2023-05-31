# Monster Arena: A Multiplayer Game Leveraging some of Algorand's capabilities

## Problem Statement

*Monster Arena* _(patent pending)_ is a multiplayer game that harnesses the power of the Algorand blockchain to foster player interactions within the MA world and its captivating creatures. Adventurers may embark on thrilling quests, vanquishing monsters scattered throughout the map using their infinite-range attacks, all while discovering and collecting exclusive NFTs dropped by their perished foes. 
Once the warriors take their prize, they must hastily make their way to the safe zone. Caution is paramount, for the night is dark and full of other hunters who, turned into opportunistic thieves, will not hesitate to leave our victors with naught but implausible tales of their feats that very few would believe.[^1]
You are tasked with building an on-chain system to store and manage monster and player's data and perform some of the basic in-game actions.


## Data layout
Here we specify the correct layout for structuring relevant data in the program.
There's two separate entities: players and monsters.
They both have locations in the map, as well as the possibility to hold an asset "in their hands" (in case of the player, modelled by their local state variable "UNSECURED_ASSET"). Players also have an unsigned integer representing their score.

An entity of type MONSTER is represented by a 24 byte structure with the following data, in the order presented:
uint64 POS_X, x coordinate on the map
uint64 POS_Y, y coordinate on the map
uint64 ASA_ID, unique NFT id

Monsters are stored in a box acting as a contiguous array, with the following structure:
Box name: b"MONSTERS"
Box contents: |uint64 len|uint64 M_0:POS_X |uint64 M_0:POS_Y |uint64 M_0:ASA_ID | (...) |uint64 M_{len}:ASA_ID | 0 | 0 | 0 (...up to 4096 bytes), where M_k are monsters

A player entity is represented by a 32 byte structure with the following data, in the order presented:
uint64 POS_X, x coordinate on the map
uint64 POS_Y, y coordinate on the map
uint64 UNSECURED_ASSET: NFT being held (or zero for none)
uint64 SCORE, a player's score (or 0 if the player is inactive)

Player's save data is stored in independant boxes, structured as follows:
Box name: b"decoded algorand address for P_k"
Box contents: |uint64 P_k:POS_X |uint64 P_k:POS_Y |uint64 P_k:UNSECURED_ASSET |uint64 P_k:SCORE|

And player's local state acts as their data storage when they're active, with the following key value pairs:
"POS_X": uint64 pos_x
"POS_Y": uint64 pos_y
"UNSECURED_ASSET": uint64 asa_id
"SCORE": uint64 score

Contract's global state:
"ADMIN": byte slice address, the address of the contract's creator

## Details
In order to build the requested system, you have to implement the following functions in your smart contract's approval program:

-On app creation (special case where _txn ApplicationID == 0_), the system should set a global state key by the name "ADMIN" and a byte slice value equal to the sender's address.

-setup("setup"), should only allow calls where the sender is equal to the value set as "ADMIN". Then, it should create a box called "MONSTERS" with 4096 bytes of space.

-addMonster("addMonster", uint64 pos_x, uint64 pos_y), adds a monster to the game. Positions are represented by an arbitrary tuple of unsigned 64 bit integers. The function should append the monster at the end of the occupied length of the MONSTER box. Consider that the first integer of the MONSTER box should be used for keeping track of the amount of live monsters in play, incrementing by one after a succesful call to this function. Lastly, the function should use an inner transaction to mint a unique NFT, for which the contract is a manager, clawback and freeze address.

-enterPlayer("enterPlayer"), if its the player's first time calling this function, it will add the player to the game. This entails creating a box named as the sender's address (32 byte value given by _txn Sender_), which will hold 4 uint64 integers: player's x and y position coordinates, the unsecured asset being ocassionaly held by the player, and the player's current score.
After a succesful entry, the player is awarded one point (which should reflect in their local state).
If the player was already in the game, but is opted out for some reason, then it should reset their box (overwrite it with 0s). Otherwise it should reinstate the data in the box to local state's respective keys, and afterwards clean the box as well.

-exitAndSavePlayer("exitAndSavePlayer"), if the player was active, it should add a player's local data into their respective box, and then clear their local state. Otherwise it should fail. Note that since score can't be zero, having a null score is a good signifier for an inactive player.

-playerMove("playerMove", Direction d), d \in {"UP", "DOWN", "LEFT", "RIGHT"}, should move an active player (fail f the caller is inactive) one unit in the desired direction, impacting their relevant local state position variable. Axis are Y+1 for up, Y-1 for down, X-1 for left and X+1 for right. Any calls that would take the player out of the bounds of unsigned integers should naturally fail.

-playerKillMonster("playerKillMonster"), should allow the caller to kill a monster anywhere on the map ("infinitely ranged attacks"), taking their NFT in the process. This means that:
    -The slot where the monster was should be filled out by whichever monster is last in the "MONSTERS" box (and the length decreased). If it was the only monster on field, it should be overwriten with zeros.
    -The ASA id of the slain monster's NFT should be filled into the player's local state variable. The NFT should then be transfered through an inner transaction. Note that the contract retains manager, freeze and clawback privileges over it, and the asset remains frozen so it can't be traded.
    -Note that when a player is holding an asset this way, their hands are "busy". This means they cannot attack other monsters, or steal from other players until they loose it (either by securing it or getting it stolen).
    -This action increases player's score by 1 point.

-pvpSteal("pvpSteal"), the caller attempts to steal a victim's unsecured prize (first available account in the Accounts array, accessed like _txna Accounts 0_). If succesful, the victim's "UNSECURED_ASSET" local variable is cleared out and the caller's is filled out with the corresponding ASA id. Then the contract claws the asset back from the victim and transfers it to the caller.
    -Players may only steal from other players holding assets at a distance of at most 10 units.
    -This action may not be carried away by a player holding an unsecured asset, as their "hands are full"

-secureAsset("secureAsset"), if the caller is holding an asset in their local state, and they are standing inside (edges included) the *safe zone* (defined as an 11 by 11 square with its lower left corner on (0,0) and its upper right corner on (10,10)), the player gets their local state variable cleared out, and their asset is safe. 
    -Only active players may secure assets.
    -Players might steal from each other inside the safe zone, if the asset has not been secured fast enough.
    -Calls without an asset to secure should fail.
    -This action increases player's score by 1 point.
    [^2]

All calls to the contract will be made through the provided script. You need not to consider any other calling structure (e.g. group transactions other than the ones utilized).
Any calls not conforming to this scheme should fail.
The contract should have global and local schema exactly as specified (or as shown in the app deployment call)


## Deployment and testing script
You will be provided with a python script, _AppTestAndDeploy.py_, that contains:
-code to compile and deploy a teal contract into a sandbox instance (algokit is recommended for the sandbox environment)
-a setup to run a series of tests for all required functionality
-some helper functions to extract and interpret values from the chain
Any edits you require can be made to this file, however remember to revert any changes and re-run tests before submitting. An edited script could invalidate your solution.


## Submitting a solution
A valid submission consists of a github repository, forked from the official hackathon repo, with your implemented TEAL code and the *unedited* latest version of _AppTestAndDeploy.py_. 
You may write your solution directly in TEAL, or use Pyteal, Beaker, Reach, or any other languages that can be compiled into TEAL.
It is not necessary to include any scripts that you used to get the final TEAL approval program, just the plain TEAL itself.

You'll be provided with a spreadsheet to input your name / team's name, repo, commit and score.
You may submit more than once, by simply editing your submission in the sheet. Keep in mind that in case of a score tie, we'll use commit timestamps to break it.


## Other details
-You may assume that no more than 170 monsters will be alive at all times (that is to say, they fit in a 4k box)
-When a player is active, their save box is zeroed out. When a player is inactive, their local state is zeroed out.
-Note that a player's score may never be 0, since they get 1 point for starting the game.
-Note that rekeying is not supported, an address change means losing your saved data.

-A solution that does not compile or deploy in a running, out of the box instance of an algokit localnet sandbox gets 0 points.
-All tests are worth the same. Points that a solution makes are equal to the amount of tests passed.
-Keep an eye open for any updates to this document and/or the deployment and testing script, extra tests might be added through the next days until friday. Also, hints / useful code snippets might be published throughout the week, so don't get discouraged if you can't figure out how to make something work.

-Ask as many questions as you want, and remember to have fun!





[^1]: This tragic twist of fate often prompts introspection, causing warriors to question their chosen path as monster hunters and to contemplate whether it was a good career choice to begin with.

[^2]: The implementation as described does allow for a player to stay comfortably in the safe zone and shoot away at monsters from there. However this strategy would kill any adventurer out of sheer embarrasment faster than any monster could.