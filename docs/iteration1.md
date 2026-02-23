#

##

###

1. studio dev tool
2. domain namespace, w boundary client
3. user from req 
4. set array in queryKey client

#### the minnimum distributed

https://www.prisma.io/docs/guides/frameworks/react-router-7

```sh
pnpm add prisma tsx @types/pg --save-dev
pnpm add @prisma/client @prisma/adapter-pg dotenv pg
pnpm dlx prisma init --db --output ../app/generated/prisma
```

[console.prisma.io] 
[github]https://github.com/fallowshades/sprints/blob/main/typescript/homeAway-next/iteration2-profile%20crud-cycle.md

```sh
pnpm dlx prisma@latest init --db
```

[prisma.db]https://www.prisma.io/docs/prisma-postgres/from-the-cli


```sh
npx prisma studio
...or online in Console:
https://console.prisma.io/cmdygz5mn056cz70vidr8ixj6/cmlze5vc001oc1kef0z6lxt1d/cmlze5vc001oa1kefjwkk0oyi/studio
```

#### entry.server namespace boundary

```ts
/**
 * Helper utility used to extract the domain from the request even if it's
 * behind a proxy. This is useful for sitemaps and other things.
 * @param request Request object
 * @returns Current domain
 */
export const createDomain = (request: Request) => {
	const headers = request.headers
	const maybeProto = headers.get("x-forwarded-proto")
	const maybeHost = headers.get("host")
	const url = new URL(request.url)
	// If the request is behind a proxy, we need to use the x-forwarded-proto and host headers
	// to get the correct domain
	if (maybeProto) {
		return `${maybeProto}://${maybeHost ?? url.host}`
	}
	// If we are in local development, return the localhost
	if (url.hostname === "localhost") {
		return `http://${url.host}`
	}
	// If we are in production, return the production domain
	return `https://${url.host}`
}
```

supabase.ts

```ts
import { createClient } from "@supabase/supabase-js"
import type { db } from "~/db.server"

// biome-ignore lint/nursery/noProcessEnv: <explanation>
const polyEnv = typeof process !== "undefined" ? process.env : window.env

const { SUPABASE_KEY, SUPABASE_URL } = polyEnv

export const supabase = createClient<typeof db>(SUPABASE_URL, SUPABASE_KEY)
```

#### get user from request


queries/user.server.ts

```ts
import { db } from "~/db.server"
import { getServerSession } from "~/session.server"

export const getUserFromRequest = async (request: Request) => {
	const session = await getServerSession(request.headers.get("Cookie"))
	const userId = session.get("id")
	if (!userId) return null
	const user = await db.player.findFirst({
		where: {
			id: userId ?? 0,
		},
	})
	return user
}
```

#### push and pull query keys

realtime/room.ts

```ts
import type { RealtimeMessage } from "@supabase/supabase-js"
import { queryClient } from "~/root"
import type { RoomLoaderData } from "~/routes/rooms.$roomId"

export const handleRoomUpdate = (roomId: string) => (payload: RealtimeMessage["payload"]) => {
	const newRoomInfo = payload.new
	// biome-ignore lint/style/noNonNullAssertion: This will be there
	const data = queryClient.getQueryData<RoomLoaderData>(["room", roomId])!
	if (payload.new.id !== data.room.id) return
	queryClient.setQueryData<typeof data>(["room", data.room.id], {
		...data,
		room: {
			...data.room,
			...newRoomInfo,
			flippedIndices: newRoomInfo.flippedIndices ?? data.room.flippedIndices,
			matchedPairs: newRoomInfo.matchedPairs ?? data.room.matchedPairs,
			currentTurn: newRoomInfo.current_turn,
		},
	})
}
```


player.ts

```ts
import type { RealtimeMessage } from "@supabase/supabase-js"
import { queryClient } from "~/root"
import type { RoomLoaderData } from "~/routes/rooms.$roomId"

export const handlePlayerUpdate =
	(roomId: string, serverLoader: () => Promise<RoomLoaderData>) => async (payload: RealtimeMessage["payload"]) => {
		const newItem = payload.new as {
			player_id: string
			isActive: boolean
			score: number
			id: string
			scrollPosition: {
				x: number
				y: number
			}
		}
		// biome-ignore lint/style/noNonNullAssertion: This will be there
		const data = queryClient.getQueryData<RoomLoaderData>(["room", roomId])!
		if (!data.players.find((p) => p.playerId === newItem.player_id)) {
			const data = await serverLoader()

			return queryClient.setQueryData(["room", roomId], data)
		}
		queryClient.setQueryData<typeof data>(["room", roomId], {
			...data,
			players: data.players.map((player) => {
				if (player.playerId === newItem.player_id) {
					return {
						...player,
						isActive: newItem.isActive,
						score: newItem.score,
						scrollPosition: newItem.scrollPosition,
					}
				}
				return player
			}),
		})
	}
```